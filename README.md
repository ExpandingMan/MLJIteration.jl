# MLJIteration.jl

| Linux | Coverage |
| :-----------: | :------: |
| [![Build status](https://github.com/ablaom/MLJIteration.jl/workflows/CI/badge.svg)](https://github.com/ablaom/MLJIteration.jl/actions)| [![codecov.io](http://codecov.io/github/ablaom/MLJIteration.jl/coverage.svg?branch=master)](http://codecov.io/github/ablaom/MLJIteration.jl?branch=master) |

A package for wrapping iterative models provided by the
[MLJ](https://alan-turing-institute.github.io/MLJ.jl/dev/) machine
learning framework in a control strategy.

Builds on the generic iteration control tool
[IterationControl.jl](https://github.com/ablaom/IterationControl.jl).

Not registered  and under construction.


## Installation

```julia
using Pkg
Pkg.add(url="https://github.com/ablaom/MLJIteration.jl")
```

## Usage

```julia
using MLJ
using MLJIteration
```

Do `?IteratedModel` for details.

What follows is a draft of documentation to be added to the [MLJ
manual](https://alan-turing-institute.github.io/MLJ.jl/dev/).

---
# Controling Iterative Models

Iterative supervised machine learning models are usually trained until
an out-of-sample estimate of the performance satisfies some stopping
criterion, such as `k` consecutive deteriorations of the performance
(see [`Patience`](@ref) below). A more sophisticated kind of control
might dynamically mutate parameters, such as a learning rate, in
response to the behaviour of these estimates. Some iterative model
implementations enable some form of automated control, with the method
and options for doing so varying from model to model. But sometimes it
is up to the user to arrange control, which in the crudest case
reduces to manually experimenting with the iteration parameter.

In response to this ad hoc state of affairs, MLJ provides a uniform
and feature-rich interface for controlling any iterative model that
exposes its iteration parameter as a hyper-parameter, and which
implements the "warm restart" behaviour described in [Machines](@ref).


## Basic use

As in [Tuning models](@ref), iteration control in MLJ is implemeted as
a model wrapper, which allows composition with other meta-algorithms.
Ordinarily, the wrapped model behaves just like the original model,
but with the training occuring on a subset of the provided data (to
allow computation of an out-of-sample loss) and with the iteration
parameter automatically determined by the controls specified in the
wrapper.

By setting `retrain=true` one can ask that the wrapped model retrain
on *all* supplied data, after learning the appropriate number of
iterations from the controlled training phase:

```@example gree
using MLJ
using MLJIteration

X, y = make_moons(1000, rng=123)
EvoTreeClassifier = @load EvoTreeClassifier verbosity=0

iterated_model = IteratedModel(model=EvoTreeClassifier(rng=123, η=0.005),
                               resampling=Holdout(rng=123),
                               measures=log_loss,
                               controls=[Step(5),
                                         Patience(2),
                                         NumberLimit(100)],
                               retrain=true)

mach = machine(iterated_model, X, y) |> fit!;
```

As detailed under [`IteratedModel`](@ref) below, the specified
`controls` are repeatedly applied in sequence to a *training machine*,
constructed under the hood, until one of the controls triggers a
stop. Here `Step(5)` means "Compute 5 more iterations" (in this case
starting from none); `Patience(2)` means "Stop at the end of the
control cycle if there have been 2 consecutive drops in the log loss";
and `NumberLimit(100)` is a safeguard ensuring a stop after 100
control cycles (500 iterations). See [Controls provided](@ref) below
for a complete list.

Because iteration is implemented as a wrapper, the "self-iterating"
model can be evaluated using cross-validation, say, and the number of
iterations on each fold will generally be different:

```@example gree
e = evaluate!(mach, resampling=CV(nfolds=3), measure=log_loss, verbosity=0);
map(e.report_per_fold) do r
    r.n_iterations
end
```

Alternatively, one might wrap the self-iterating model in a tuning
strategy, using `TunedModel`; see [Tuning Models](@ref). In this way,
the optimization of some other hyper-parameter is realized
simultaneously with that of the iteration parameter, which will
frequently be more efficient than a direct two-parameter search.


## Using training losses, and controlling model tuning

Some iterative models report a training loss, as a biproduct of a
`fit!` call, and these can be used in two ways:

1. To supplement an out-of-sample estimate of the loss in deciding
 when to stop, as in the `PQ` stopping criterion (see [Prechelt, Lutz
 (1998)](https://link.springer.com/chapter/10.1007%2F3-540-49430-8_3)));
 or

2. As a (generally less reliable) substitute for an out-of-sample
  loss, when wishing to train on all data supplied data.

To have `IteratedModel` bind all data to the training machine and use
training losses in place of an out-of-sample loss, simply specify
`resampling=nothing`. To check if `MyFavoriteIterativeModel` reports
training losses, load the model code and inspect
`supports_training_losses(MyFavoriteIterativeModel)` (or do
`info("MyFavoriteIterativeModel")`)


### Model tuning

An example of scenario 2 occurs when controlling hyper-parameter
optimization (model tuning). Recall that MLJ's [`TunedModel`](@ref)
wrapper is implemented as an iterative model. Moreover, this wrapper
reports, as a training loss, the lowest value of the optimization
objective function so far (typically the lowest value of an
out-of-sample loss, or -1 times an out-of-sample score). One may want
to simply end the hyper-parameter search when this value meets the
[`NumberSinceBest`](@ref) stopping criterion discussed below, say,
rather than introduce an extra layer of resampling to first "learn"
the optimal value of the iteration parameter.

In the following example we conduct a [`RandomSearch`](@ref) for the
optimal value of the regularizaion parameter `lambda` in a
`RidgeRegressor` using 6-fold cross-validation. By wrapping our
"self-tuning" version of the regressor as an [`IteratedModel`](@ref),
with `resampling=nothing` and `NumberSinceBest(20)` in the controls,
we terminate the search when the number of `lambda` values tested
since the previous best cross-validation loss reaches 20.

```@example gree
using MLJ
using MLJIteration

X, y = @load_boston;
RidgeRegressor = @load RidgeRegressor pkg=MLJLinearModels verbosity=0
model = RidgeRegressor()
r = range(model, :lambda, lower=-1, upper=2, scale=x->10^x)
self_tuning_model = TunedModel(model=model,
                               tuning=RandomSearch(rng=123),
                               resampling=CV(nfolds=6),
                               range=r,
                               measure=mae);
iterated_model = IteratedModel(model=self_tuning_model,
                               resampling=nothing,
                               control=[Step(1), NumberSinceBest(20), NumberLimit(1000)])
mach = machine(iterated_model, X, y);
fit!(mach);
report(mach).model_report.best_model
```

We can use `mach` here to directly obtain predictions using the
optimal model, as in

```@example gree
predict(mach, selectrows(X, 1:4))
```

## Controls provided

In the table below, `mach` is the *training machine* being iterated,
constructed by binding the supplied data to the `model` specified in
the `IteratedModel` wrapper, but trained in each iteration on a subset
of the data, according to the value of the `resampling`
hyper-parameter of the wrapper.


control                                              | description                                                                             | can trigger a stop
-----------------------------------------------------|-----------------------------------------------------------------------------------------|--------------------
[`Step`](@ref)`(n=1)`                                | Train model for `n` more iterations                                                     | no
[`TimeLimit`](@ref)`(t=0.5)`                         | Stop after `t` hours                                                                    | yes
[`NumberLimit`](@ref)`(n=100)`                       | Stop after `n` applications of the control                                              | yes
[`NumberSinceBest`](@ref)`(n=6)`                           | Stop when best loss occurred `n` control applications ago                               | yes
[`NotANumber`](@ref)`()`                             | Stop when `NaN` encountered                                                             | yes
[`Threshold`](@ref)`(value=0.0)`                     | Stop when `loss < value`                                                                | yes
[`GL`](@ref)`(alpha=2.0)`                            | ★ Stop after "GeneralizationLossDo" exceeds `alpha`                                     | yes
[`Patience`](@ref)`(n=5)`                            | ★ Stop after `n` consecutive loss increases                                             | yes
[`PQ`](@ref)`(alpha=0.75, k=5)`                      | ★ Stop after "Progress-modified GL" exceeds `alpha`                                     | yes
[`Info`](@ref)`(f=identity)`                         | Log to `Info` the value of `f(mach)`, where `mach` is current machine                   | no
[`Warn`](@ref)`(predicate; f="")`                    | Log to `Warn` the value of `f` or `f(mach)` if `predicate(mach)` holds                  | no
[`Error`](@ref)`(predicate; f="")`                   | Log to `Error` the value of `f` or `f(mach)` if `predicate(mach)` holds and then stop   | yes
[`Callback`](@ref)`(f=_->nothing)`                   | Call `f(mach)`                                                                          | yes
[`WithNumberDo`](@ref)`(f=n->@info(n))`              | Call `f(n + 1)` where `n` is number of previous calls                                   | yes
[`WithIterationsDo`](@ref)`(f=x->@info("loss: $x"))` | Call `f(i)`, where `i` is total number of iterations                                    | yes
[`WithLossDo`](@ref)`(f=x->@info(x))`                | Call `f(loss)` where `loss` is the current loss                                         | yes
[`WithTrainingLossesDo`](@ref)`(f=v->@info(v))`      | Call `f(v)` where `v` is the current batch of training losses                           | yes
[`Save`](@ref)`(filename="machine.jlso")`            | Save current machine to `machine1.jlso`, `machine2.jslo`, etc                           | yes

> Table 1. Atomic controls. Some advanced options omitted.

★ For more these controls see [Prechelt, Lutz
 (1998)](https://link.springer.com/chapter/10.1007%2F3-540-49430-8_3):
 "Early Stopping - But When?", in *Neural Networks: Tricks of the
 Trade*, ed. G. Orr, Springer.

**Stopping option.** All the following controls trigger a stop if the
provided function `f` returns `true` and `stop_if_true=true` is
specified in the constructor: `Callback`, `WithNumberDo`,
`WithLossDo`, `WithTrainingLossesDo`.

There are also three control wrappers to modify a control's behavior:

wrapper                                            | description
---------------------------------------------------|-------------------------------------------------------------------------
`IterationControl.skip(control, predicate=1)`      | Apply `control` every `predicate` applications of the control wrapper (can also be a function; see doc-string)
`IterationControl.debug(control)`                  | Apply `control` but also log its state to `Info` (irrespective of `verbosity` level)
`IterationControl.composite(controls...)`          | Apply each `control` in `controls` in sequence; mostly for under-the-hood use

> Table 2. Wrapped controls

```@docs
Step
TimeLimit
NumberLimit
NumberSinceBest
NotANumber
Threshold
GL
PQ
Info
Warn
Error
Callback
WithNumberDo
WithIterationsDo
WithLossDo
WIthTrainingLossesDo
Save
```

## Custom controls

Control in MLJIteration is implemented using
[IterationControl.jl](https://github.com/ablaom/IterationControl.jl). Rather
than iterating a training machine directly, we iterate a wrapped
version of this object, which includes other information that controls may
want to access, such the MLJ evaluation object. This information is
summarized under [The training machine wrapper](@ref) below.

Controls must implement two `update!` methods, one for initializing
the control's *state* on the first application of the control (this
state being external to the control `struct`) and one for all
subsequent control applications, which generally updates state
also. There are two optional methods: `done`, for specifying
conditions triggering a stop, and `takedown` for specifying actions to
perform at the end of all training.

We summarize the training algorithm, as it relates to controls, after
giving a simple example.


### Example 1 - Non-uniform iteration steps

Below we define  a control, `IterateFromList(list)`, to  train, on the
each application of the control, until the iteration count reaches the
next value on  a user-specified list, triggering a stop  when the list
is    exhausted.     So,    for    example,    we     might    specify
`IterateFromList(2:2:100)`, which  would amount to specifying  the two
existing controls `Step(2)` and `NumberLimit(100)`.

In the code, `wrapper` is an object that wraps the training machine
(see above).

```julia
struct IterateFromList
    list::Vector{<:Int} # list of iteration parameter values
    IterateFromList(v) = new(unique(sort(v)))
end

function MLJIteration.update!(control::IterateFromList, wrapper, verbosity)
    Δi = control.list[1]
    verbosity > 1 && @info "Training $Δi more iterations. "
    MLJIteration.train!(wrapper, Δi) # trains the training machine
    return (index = 2, )
end

function MLJIteration.update!(control::IterateFromList, wrapper, verbosity, state)
    index = state.positioin_in_list
    Δi = control.list[i] - wrapper.n_iterations
    verbosity > 1 && @info "Training $Δi more iterations. "
    MLJIteration.train!(wrapper, Δi)
    return (index = index + 1, )
end
```

The first `update` method will be called the first time the control is
applied, returning an initialized `state = (index = 2,)`, which is
passed to the second `update` method, which is called on subsequent
control applications (and which returns the updated `state`).

A `done` method articulates the criterion for stopping:

```julia
MLJIteration.done(control::IterateFromList, state) =
    state.index > length(control.list)
```

For the sake of illustration, we'll implement a `takedown` method; its
return value is included in the `IteratedModel` report:

```julia
MLJIteration.takedown(control::IterateFromList, verbosity, state) 
    verbosity > 1 && = @info "Stepped through these values of the "*
                              "iteration parameter: $(control.list)"
    return (iteration_values=control.list, )
end
```


### The training algorithm

Here now is a simplified description of the training of an
`IteratedModel`. First, the atomic `model` is bound to a subset of the
supplied data and wrapped in an object called `wrapper` below. To
train the wrapped machine for `i` more iterations, and update the
other data in the wrapper, requires the call
`MLJIteration.train!(wrapper, i)`. Only controls make this call (e.g.,
`Step(...)`, or `IterateFromList` above). If we assume for simplicity
there is only a single control, called `control`, then training
proceeds as follows:

```julia
state = update!(control, wrapper, verbosity)
finished = done(control, state)

# subsequent training events:
while !finished
    state = update!(control, wrapper, verbosity, state)
    finished = done(control, state)
end

# finalization:
return takedown(control, verbosity, state)
```

### The training machine wrapper

A training machine `wrapper` has these properties:

- `wrapper.machine` - the training machine, type `Machine`

- `wrapper.model`   - the mutable atomic model, coinciding with `wrapper.machine.model`

- `wrapper.n_cycles` - the number `IterationControl.train!(wrapper, _)` calls
  so far; generally the current control cycle count
  
- `wrapper.n_iterations` - the total number of iterations applied to the model so far

- `wrapper.Δiterations` - the number of iterations applied in the last
  `IterationControl.train!(wrapper, _)` call
  
- `wrapper.loss` - the out-of-sample loss (based on the first measure in `measures`)

- `wrapper.training_losses` - the last bactch of training losses (if
  reported by `model`), an abstract vector of length
  `wrapper.Δiteration`.
  
- `wrapper.evalution` - the complete MLJ performance evaluation
  object, which has the following properties: `measure`,
  `measurement`, `per_fold`, `per_observation`,
  `fitted_params_per_fold`, `report_per_fold` (here there is only one
  fold). For further details, see [Evaluating Model Performance](@ref).


### Example 3 - Cyclic learning rates


```julia
struct CylicLearningStep{F<:AbstractFloat}
        n::Int           # number of cycles of learning rate mutations
        stepsize::Int    # twice this is the cycle period
        min_lr::F        # lower learning rate
        max_lr::F        # upper learning rate
    learning_rate_parameter::Union{Symbol,Expr}
end

```








## API

```@docs
IteratedModel
```
