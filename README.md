# SAGE

**SAGE (Shapley Additive Global importancE)** is a game-theoretic approach for understanding black-box machine learning models. It summarizes each feature's importance based on the predictive power it contributes, and it accounts for complex feature interactions using the Shapley value.

SAGE was introduced in [this paper](https://arxiv.org/abs/2004.00668), but if you're new to using Shapley values you might want to start by reading this [blog post](https://iancovert.com/blog/understanding-shap-sage/).

## Install

<!--The easiest way to use the code is to install `sage-importance` with `pip`:

```bash
pip install sage-importance
```

Alternatively, you can clone the repository and install the package using the local `setup.py` file:

```bash
pip install .
```-->

The easiest way to get started is to clone the repository and install the package into your Python environment:

```bash
pip install .
```

## Usage

SAGE is model-agnostic, so you can use it with any kind of machine learning model (linear models, GBMs, neural networks, etc). All you need to do is set up an imputer to handle held out features and run a Shapley value estimator:

```python
import sage

# Get data
x, y = ...
feature_names = ...

# Get model
model = ...

# Set up imputer for missing features
imputer = sage.MarginalImputer(model, x[:512])

# Set up estimator
estimator = sage.PermutationEstimator(imputer, 'mse')

# Calculate SAGE values
sage_values = estimator(x, y)
sage_values.plot(feature_names)
```

The result will look like this:

<p align="center">
  <img width="540" src="https://raw.githubusercontent.com/iancovert/sage/master/docs/bike.svg"/>
</p>

Our implementation supports several features to make Shapley value calculation more practical:

- **Uncertainty estimation.** Confidence intervals are provided for each feature's importance value.
- **Convergence.** Convergence is determined automatically based on the size of the confidence intervals, and a progress bar displays the estimated time until convergence.
- **Model conversion.** Our back-end requires models that are converted into a consistent format, and the conversion step is performed automatically for XGBoost, CatBoost, LightGBM, sklearn and PyTorch models. If you're using a different kind of model, it must be converted to a callable function (see [here](https://github.com/iancovert/sage/blob/master/sage/utils.py#L5) for examples).

## Examples

Check out the following notebooks to get started:

- [Bike](https://github.com/iancovert/sage/blob/master/notebooks/bike.ipynb) is a simple example using XGBoost, and it shows how to calculate SAGE values and Shapley Effects (an alternative explanation when no labels are available)
- [Credit](https://github.com/iancovert/sage/blob/master/notebooks/credit.ipynb) shows how to generate explanations with a surrogate model to approximate the conditional distribution (using CatBoost)
- [Airbnb](https://github.com/iancovert/sage/blob/master/notebooks/airbnb.ipynb) shows an example where SAGE values are calculated with grouped features (using a PyTorch MLP)
- [Bank](https://github.com/iancovert/sage/blob/master/notebooks/bank.ipynb) shows a model monitoring example that uses SAGE to identify features that hurt the model's performance (using CatBoost)
- [MNIST](https://github.com/iancovert/sage/blob/master/notebooks/mnist.ipynb) shows several strategies to accelerate convergence for datasets with many features (feature grouping, different imputing setups)

If you want to replicate any experiments described in our paper, see this separate [repository](https://github.com/iancovert/sage-experiments).

## More details

This repository provides some flexibility in the explanations that are provided. You can make several choices when generating explanations.

### 1. Feature removal approach

The original SAGE paper proposes marginalizing out missing features using their conditional distribution. Since this is challenging to implement in practice, several approximations are available. The choices include

1. Use default values for missing features (see [MNIST](https://github.com/iancovert/sage/blob/master/notebooks/mnist.ipynb) for an example). This is a fast but low-quality approximation.
2. Sample features from the marginal distribution (see [Bike](https://github.com/iancovert/sage/blob/master/notebooks/bike.ipynb) for an example). This approximation is discussed in the SAGE paper.
3. Train a supervised surrogate model (see [Credit](https://github.com/iancovert/sage/blob/master/notebooks/credit.ipynb) for an example). This approach is described in this [paper](https://arxiv.org/abs/2011.14878), and it can provide a better approximation than the other approaches. However, it requires training an additional model (typically a neural network).

### 2. Explanation type

Two types of explanations can be calculated, both based on Shapley values:

1. **SAGE.** This approach quantifies each feature's role in improving the model's performance (the default explanation here).
2. **Shapley Effects.** Described in this [paper](https://epubs.siam.org/doi/pdf/10.1137/130936233?casa_token=fU5qvdv35pkAAAAA:jlQsuRWlPrZ5j3YgaPdOmgOV2-B7FnWB5arog_wj4Sqo4OBTuZsHEgJRPGO7vR1D0UOH8-t9UHU), this explanation method quantifies the model's sensitivity to each feature. Since Shapley Effects is a variation on SAGE (see details in this [paper](https://arxiv.org/abs/2011.14878)), our implementation generates this type of explanation *when labels are not provided*. See the [Bike](https://github.com/iancovert/sage/blob/master/notebooks/bike.ipynb) notebook for an example.

### 3. Shapley value estimator

Shapley values are computationally costly to calculate, so we implemented four different estimators:

1. **Permutation sampling.** This is the approach described in the original paper (see `PermutationEstimator`). 
2. **KernelSAGE.** This is a linear regression-based estimatorthat is similar to KernelSHAP (see `KernelEstimator`). It is described in this [paper](https://arxiv.org/abs/2012.01536), and the [Bank](https://github.com/iancovert/sage/blob/master/notebooks/bank.ipynb) notebook shows an example use-case.
3. **Iterated sampling.** This is a variation on the permutation sampling approach where we calculate Shapley values for each feature sequentially (see `IteratedEstimator`). This permits faster convergence for features with low variance, but it can result in wider confidence intervals.
4. **Sign estimation**. This method estimates SAGE values to a lower precision by focusing on their sign (i.e., whether they help or hurt performance). It is implemented in `SignEstimator`, and the [Bank](https://github.com/iancovert/sage/blob/master/notebooks/bank.ipynb) notebook shows an example.

The results from each approach should be identical because they are all unbiased estimators. However, their convergence speed may differ. Permutation sampling is a good approach to start with. KernelSAGE converges a bit faster, but the uncertainty is spread more evenly among the features (rather than being highest for more important features).

### 4. Grouped features

Rather than removing features individually, you can specify groups of features to be removed together. This will likely speed up convergence because there are fewer feature subsets to consider. See the [Airbnb](https://github.com/iancovert/sage/blob/master/notebooks/airbnb.ipynb) notebook for an example.

## Authors

- Ian Covert (<icovert@cs.washington.edu>)
- Scott Lundberg
- Su-In Lee

## References

Ian Covert, Scott Lundberg, Su-In Lee. "Understanding Global Feature Contributions With Additive Importance Measures." *NeurIPS 2020*

Ian Covert, Scott Lundberg, Su-In Lee. "Explaining by Removing: A Unified Framework for Model Explanation." *arxiv preprint:2011.14878*

Ian Covert, Su-In Lee. "Improving KernelSHAP: Practical Shapley Value Estimation via Linear Regression." AISTATS 2021

Art Owen. "Sobol' Indices and Shapley value." *SIAM 2014*
