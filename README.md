# nyaggle
![GitHub Actions CI Status](https://github.com/nyanp/nyaggle/workflows/Python%20package/badge.svg)
![Python Versions](https://img.shields.io/pypi/pyversions/nyaggle.svg?logo=python&logoColor=white)

**nyaggle** is a utility library for Kaggle and offline competitions, 
particularly focused on experiment logging, feature engineering and validation. 

- [documentation](https://nyaggle.readthedocs.io/en/latest/index.html)
- [slide (Japanese)](https://docs.google.com/presentation/d/1jv3J7DISw8phZT4z9rqjM-azdrQ4L4wWJN5P-gKL6fA/edit?usp=sharing)

## Installation
You can install nyaggle via pip:
```
$pip install nyaggle
```

## Examples

### Experiment Logging
`run_experiment()` is an high-level API for experiment with cross validation.
It outputs parameters, metrics, out of fold predictions, test predictions,
feature importance and submission.csv under the specified directory.

It can be combined with mlflow tracking.

```python
from sklearn.model_selection import train_test_split

from nyaggle.experiment import run_experiment
from nyaggle.testing import make_classification_df

X, y = make_classification_df()
X_train, X_test, y_train, y_test = train_test_split(X, y)

params = {
    'n_estimators': 1000,
    'max_depth': 8
}

result = run_experiment(params,
                        X_train,
                        y_train,
                        X_test)
                         
# You can get outputs that needed in data science competitions with 1 API

print(result.test_prediction)  # Test prediction in numpy array
print(result.oof_prediction)   # Out-of-fold prediction in numpy array
print(result.models)           # Trained models for each fold
print(result.importance)       # Feature importance for each fold
print(result.metrics)          # Evalulation metrics for each fold
print(result.time)             # Elapsed time
print(result.submission_df)    # The output dataframe saved as submission.csv

# ...and all outputs have been saved under the logging directory (default: output/yyyymmdd_HHMMSS).


# You can use it with mlflow and track your experiments through mlflow-ui
result = run_experiment(params,
                        X_train,
                        y_train,
                        X_test,
                        with_mlflow=True)
```


### Feature Engineering

#### Target Encoding with K-Fold
```python
import pandas as pd
import numpy as np

from sklearn.model_selection import KFold
from nyaggle.feature.category_encoder import TargetEncoder


train = pd.read_csv('train.csv')
test = pd.read_csv('test.csv')
all = pd.concat([train, test]).copy()

cat_cols = [c for c in train.columns if train[c].dtype == np.object]
target_col = 'y'

kf = KFold(5)

# Target encoding with K-fold
te = TargetEncoder(kf.split(train))

# use fit/fit_transform to train data, then apply transform to test data
train.loc[:, cat_cols] = te.fit_transform(train[cat_cols], train[target_col])
test.loc[:, cat_cols] = te.transform(test[cat_cols])

# ... or just call fit_transform to concatenated data
all.loc[:, cat_cols] = te.fit_transform(all[cat_cols], all[cat_cols])
```

#### Text Vectorization using BERT
You need to install pytorch to your virtual environment to use BertSentenceVectorizer. 
MaCab and mecab-python3 are also required if you use Japanese BERT model.

```python
import pandas as pd
from nyaggle.feature.nlp import BertSentenceVectorizer


train = pd.read_csv('train.csv')
test = pd.read_csv('test.csv')
all = pd.concat([train, test]).copy()

text_cols = ['body']
target_col = 'y'
group_col = 'user_id'


# extract BERT-based sentence vector
bv = BertSentenceVectorizer(text_columns=text_cols)

text_vector = bv.fit_transform(train)


# BERT + SVD, with cuda
bv = BertSentenceVectorizer(text_columns=text_cols, use_cuda=True, n_components=40)

text_vector_svd = bv.fit_transform(train)

# Japanese BERT
bv = BertSentenceVectorizer(text_columns=text_cols, lang='jp')

japanese_text_vector = bv.fit_transform(train)
```
