# Supervised training using [tabular-transformer](https://github.com/echosprint/TabularTransformer)

This notebook demonstrates how to use the TabularTransformer to efficiently handle a tabular data prediction task. We will walk through data preparation, model training, and prediction using a sample dataset.

TabularTransformer is an end-to-end training framework that processes raw data directly, eliminating the need for handcrafted features and complex preprocessing steps. It provides a competitive alternative to tree-based models for handling tabular data.

<a target="_blank" href="https://colab.research.google.com/github/echosprint/TabularTransformer/blob/main/notebooks/supervised_training.ipynb">
  <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
</a>

Alternatively, you can check out the [notebook](https://github.com/echosprint/TabularTransformer/blob/main/notebooks/supervised_training.ipynb) to run it locally.

Please note that the hyperparameters used here are not optimized and may be suboptimal.

## Setup
First, we need to install the `tabular-transformer` package.


```python
%pip install tabular-transformer
```

Next, we import the necessary modules.


```python
import tabular_transformer as ttf
import torch
```

## Data Preparation

We will use the [Adult Income dataset](https://huggingface.co/datasets/scikit-learn/adult-census-income) for our prediction task, which aims to determine whether a person makes over $50K a year. The dataset contains 32.6k rows and 15 columns, with the label column named `income`.

```python
income_dataset_path = ttf.prepare_income_dataset()
```

TabularTransformer treats each column feature as either `Categorical` or `Numerical`. Here, we need to specify which columns fall into each category. For convenience, one of the lists can be left empty. In the snippet below, `numerical_cols` will includes all columns that are not specified as categorical.


```python
categorical_cols = [
    'workclass', 'education',
    'marital.status', 'occupation',
    'relationship', 'race', 'sex',
    'native.country', 'income']

# all remaining columns are numerical
numerical_cols = []
```

To properly instruct `tabular-transformer` on how to handle the data, we define an instance of the `DataReader`. This instance specifies which columns are categorical, which are numerical, and handles other data-related settings such as file paths, label column, header presence, ID column, etc. 


```python
income_reader = ttf.DataReader(
    file_path=income_dataset_path,
    ensure_categorical_cols=categorical_cols,
    ensure_numerical_cols=numerical_cols,
    label='income',
    header=True,
    id=None,
)
```


Optionally, we can split the data into training and testing sets if a testing set is not already available. This dataset contains a mix of numerical, categorical, and missing features. `tabular-transformer` processes the data as is, so no preprocessing is required. Here, we split 20% of the data as the testing set and use the remaining 80% for training.


```python
split = income_reader.split_data(
    split={'test': 0.2, 'train': -1},
    seed=42,
    output_path='data/income/',
    save_as='csv',
)
split
```

we create data readers for both the training and testing datasets using the `income_reader` instance.

The `ttf.DataReader` instance is callable object, can be called to update specific attributes and return the updated instance. 


```python
train_data_reader = income_reader(file_path=split['train'])
test_data_reader = income_reader(file_path=split['test'])
```

## Model Training

Next, we specify the device for computation (CPU or GPU) and the data type to be used for training. We detect whether CUDA is available for acceleration; if not, we default to CPU.

```python
device = 'cuda' if torch.cuda.is_available() else 'cpu'
dtype = 'bfloat16' if torch.cuda.is_available() \
    and torch.cuda.is_bf16_supported() else 'float16'
```

We set up the training settings using `ttf.TrainSettings`. These settings define various configurations that guide the model's training behavior:

```python
ts = ttf.TrainSettings(
    device=device,
    dtype=dtype,
    apply_power_transform=True,
    min_cat_count=0.02,
)
```

Explanation of Parameters:

- device: Specifies the device (CPU or GPU) on which the training will be executed. Using a GPU (if available) can significantly speed up the training process.

- dtype: Defines the data type (e.g., torch.float32, torch.bfloat16) used during training, which can impact memory usage and computational performance.

- apply_power_transform: Indicates whether a power transformation should be applied to numerical features. This can help stabilize variance, making the data more suitable for training.

- min_cat_count: Sets the minimum count (as a proportion) for categorical values. Categories occurring less frequently than this threshold will be labeled as unknown, which helps in managing rare categories effectively.


Next, we define the model's hyperparameters using `ttf.HyperParameters`. These parameters determine the architecture and capacity of the TabularTransformer model:

```python
hp = ttf.HyperParameters(
    dim=64,
    n_heads=8,
    n_layers=6)
```
Explanation of Hyperparameters:

- dim: Sets the dimensionality of the embeddings in the Transformer. A value of 64 indicates that each embedding vector will have 64 dimensions, influencing the model's capacity to capture complex patterns in the data.

- n_heads: Specifies the number of attention heads in each multi-head attention layer. Using 8 heads allows the model to focus on different parts of the input sequence simultaneously, capturing diverse relationships within the data.

- n_layers: Defines the number of layers (or blocks) in the Transformer. Here, 6 layers provide the depth necessary for the model to learn intricate patterns in tabular data through sequential processing.


We then define the training parameters for the TabularTransformer model using `ttf.TrainParameters`. Each parameter is crucial in guiding the model's training process:


```python
tp = ttf.TrainParameters(
    max_iters=3000, learning_rate=5e-4,
    output_dim=1, loss_type='BINCE',
    batch_size=128, eval_interval=100,
    eval_iters=20, warmup_iters=100,
    validate_split=0.2)
```

Explanation of Parameters:

- max_iters: Sets the maximum number of iterations for training, determining how many times the model will go through the training data.

- learning_rate: Controls the step size during model weight updates. A smaller value like 5e-4 ensures the model learns gradually, reducing the risk of overshooting the optimal solution.

- output_dim: Specifies the dimension of the model's output. Here, it is set to 1, which is typical for binary classification or regression tasks.

- loss_type: Indicates the loss function to be used during training. `BINCE` stands for Binary Cross-Entropy, commonly used for binary classification problems.

- batch_size: The number of samples processed in each iteration. A batch size of 128 balances the need for stable gradient estimates and computational efficiency.

- eval_interval: The model will be evaluated on the validation set every 100 iterations, allowing you to monitor its performance regularly during training.

- eval_iters: Defines the number of iterations used during the evaluation phase. It helps average the performance over several mini-batches to get a more stable evaluation metric.

- warmup_iters: Specifies a warm-up phase for the first 100 iterations where the learning rate gradually increases to its set value. This technique helps stabilize the initial training phase.

- validate_split: The proportion of the dataset reserved for validation. Here, 20% of the data will be used to validate the model's performance during training, ensuring that the model is not overfitting.


Finally, we create a `ttf.Trainer` instance and initiate the training process for the TabularTransformer model. We will train the model using a one-liner.


```python
trainer = ttf.Trainer(hp=hp, ts=ts)

trainer.train(
    data_reader=train_data_reader,
    tp=tp)
```

Explanation:

   - `trainer = ttf.Trainer(hp=hp, ts=ts)`: This line initializes the Trainer with the specified hyperparameters (`hp`) and training settings (`ts`). The Trainer is responsible for managing the training loop, optimization, and model evaluation.

   - `trainer.train(data_reader=train_data_reader, tp=tp)`: This line starts the training process. It takes the `train_data_reader` to load the training data and `tp` for the training parameters (e.g., learning rate, batch size). During training, the model iteratively processes the data, adjusts its weights, and learns patterns from the input features to improve its predictive performance.

## Making Predictions
After training the model, we use the trained model checkpoint to make predictions on the test data.

The code below demonstrates how to use the trained model to make predictions on the test dataset:


```python
predictor = ttf.Predictor(checkpoint='out/ckpt.pt')

prediction = predictor.predict(
    data_reader=test_data_reader,
    save_as="prediction_income.csv"
)
```

Explanation:

   - `predictor = ttf.Predictor(checkpoint='out/ckpt.pt')`: This line creates an instance of Predictor using the model checkpoint stored at 'out/ckpt.pt'. The checkpoint contains the trained model's parameters and train configurations, allowing us to make predictions on new data.

   - `prediction = predictor.predict(data_reader=test_data_reader, save_as="prediction_income.csv")`: This line runs the prediction process using the `test_data_reader` to load and process the test data. The results are saved to a CSV file named `"prediction_income.csv"`. The `predict()` method generates predictions based on the patterns the model learned during training.

 displays the first few rows of the prediction `pd.DataFrame`


```python
prediction.head(3)
```

## Conclusion
In this notebook, we demonstrated the workflow of using the TabularTransformer for handling tabular data. You can experiment with different hyperparameters and datasets to explore the model's capabilities further.
