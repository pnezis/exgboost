[![Documentation](https://img.shields.io/badge/-Documentation-blueviolet)](https://hexdocs.pm/exgboost)

# Exgboost

Elixir bindings to the [XGBoost C API](https://xgboost.readthedocs.io/en/latest/c.html) using [Native Implemented Functions (NIFs)](https://www.erlang.org/doc/man/erl_nif.html).

Exgboost is currently based off of [this](https://github.com/dmlc/xgboost/tree/08ce495b5de973033160e7c7b650abf59346a984) commit for the upcoming `2.0.0` release of XGBoost.

`Exgboost` provides an implementation of XGBoost that works with
[Nx](https://hexdocs.pm/nx/Nx.html) tensors.

Xtreme Gradient Boosting (XGBoost) is an optimized distributed gradient
boosting library designed to be highly efficient, flexible and portable.
It implements machine learning algorithms under the [Gradient Boosting](https://en.wikipedia.org/wiki/Gradient_boosting)
framework. XGBoost provides a parallel tree boosting (also known as GBDT, GBM)
that solve many data science problems in a fast and accurate way. The same code
runs on major distributed environment (Hadoop, SGE, MPI) and can solve problems beyond
billions of examples.

## Installation

```elixir
def deps do
[
  {:exgboost, "~> 0.1.0"}
]
end
```

## API Data Structures

Exgboost's top-level `Exgboost` API works directly and only with `Nx` tensors. However, under the hood,
it leverages the structs defined in the `Exgboost.Booster` and `Exgboost.DMatrix` modules. These structs
are wrappers around the structs defined in the XGBoost library.
The two main structs used are [DMatrix](https://xgboost.readthedocs.io/en/latest/c.html#dmatrix)
to represent the data matrix that will be used
to train the model, and [Booster](https://xgboost.readthedocs.io/en/latest/c.html#booster)
which represents the model.

The top-level `Exgboost` API does not expose the structs directly. Instead, the
structs are exposed through the `Exgboost.Booster` and `Exgboost.DMatrix` modules. Power users
might wish to use these modules directly. For example, if you wish to use the `Booster` struct
directly then you can use the `Exgboost.Booster.booster/2` function to create a `Booster` struct
from a `DMatrix` and a keyword list of options. See the `Exgboost.Booster` and `Exgboost.DMatrix`
modules source for more implementation details.

## Basic Usage

```elixir
key = Nx.Random.key(42)
{X, _} = Nx.Random.normal(key, 0, 1, shape: {10, 5})
{y, _} = Nx.Random.normal(key, 0, 1, shape: {10})
model = Exgboost.train(X,y)
Exgboost.predict(model, X)
```

## Training

Exgboost is designed to feel familiar to the users of the Python XGBoost library. `Exgboost.train/2` is the
primary entry point for training a model. It accepts a Nx tensor for the features and a Nx tensor for the labels.
`Exgboost.train/2` returns a trained`Booster` struct that can be used for prediction. `Exgboost.train/2` also
accepts a keyword list of options that can be used to configure the training process. See the
[XGBoost documentation](https://xgboost.readthedocs.io/en/latest/parameter.html) for the full list of options.

`Exgbost.train/2` uses the `Exgboost.Training.train/1` function to perform the actual training. `Exgboost.Training.train/1`
and can be used directly if you wish to work directly with the `DMatrix` and `Booster` structs.

One of the main features of `Exgboost.train/2` is the ability for the end user to provide a custom training function
that will be used to train the model. This is done by passing a function to the `:obj` option. The function must
accept a `DMatrix` and a `Booster` and return a `Booster`. The function will be called at each iteration of the
training process. This allows the user to implement custom training logic. For example, the user could implement
a custom loss function or a custom metric function. See the [XGBoost documentation](https://xgboost.readthedocs.io/en/latest/tutorials/custom_metric_obj.html)
for more information on custom loss functions and custom metric functions.

Another feature of `Exgboost.train/2` is the ability to provide a validation set for early stopping. This is done
by passing a list of 3-tuples to the `:evals` option. Each 3-tuple should contain a Nx tensor for the features, a Nx tensor
for the labels, and a string label for the validation set name. The validation set will be used to calculate the validation
error at each iteration of the training process. If the validation error does not improve for `:early_stopping_rounds` iterations
then the training process will stop. See the [XGBoost documentation](https://xgboost.readthedocs.io/en/latest/tutorials/param_tuning.html)
for a more detailed explanation of early stopping.

Early stopping is achieved through the use of callbacks. `Exgboost.train/2` accepts a list of callbacks that will be called
at each iteration of the training process. The callbacks can be used to implement custom logic. For example, the user could
implement a callback that will print the validation error at each iteration of the training process or to provide a custom
setup function for training. See the `Exgboost.Training.Callback` module for more information on callbacks.

Please notes that callbacks are called in the order that they are provided. If you provide multiple callbacks that modify
the same parameter then the last callback will trump the previous callbacks. For example, if you provide a callback that
sets the `:early_stopping_rounds` parameter to 10 and then provide a callback that sets the `:early_stopping_rounds` parameter
to 20 then the `:early_stopping_rounds` parameter will be set to 20.

You are also able to pass parameters to be applied to the Booster model using the `:params` option. These parameters will
be applied to the Booster model before training begins. This allows you to set parameters that are not available as options
to `Exgboost.train/2`. See the [XGBoost documentation](https://xgboost.readthedocs.io/en/latest/parameter.html) for a full
list of parameters.

```elixir
Exgboot.train(X,
              y,
              obj: &Exgboost.Training.train/1,
              evals: [{X_test, y_test, "test"}],
              learning_rates: fn i -> i/10 end,
              num_boost_round: 10,
              early_stopping_rounds: 3,
              params: [max_depth: 3, eval_metric: ["rmse","logloss"]])
```

## Prediction

`Exgboost.predict/2` is the primary entry point for making predictions with a trained model.
It accepts a `Booster` struct (which is the output of `Exgboost.train/2`).
`Exgboost.predict/2` returns a Nx tensor containing the predictions.
`Exgboost.predict/2` also accepts a keyword list of options that can be used to configure the prediction process.

```elixir
preds = Exgboost.train(X, y) |> Exgboost.predict(X)
```

# Requirements

* Make
* CMake
* If MacOS: `brew install libomp`

When you run `mix compile`, the `xgboost` shared library will be compiled, so the first time you compile your project will take longer than subsequent compilations.

## Known Limitations

The XGBoost C API uses C function pointers to implement streaming data types.  The Python ctypes library is able to pass function pointers to the C API which are then executed by XGBoost. Erlang/Elixir NIFs do not have this capability, and as such, streaming data types are not supported in Exgboost. 

## Roadmap

* [ ] Model Serialization
* [ ] CUDA support
* [ ] [Collective API](https://xgboost.readthedocs.io/en/latest/c.html#collective)?

## License

Licensed under an [Apache-2](https://github.com/acalejos/exgboost/blob/main/LICENSE) license.
