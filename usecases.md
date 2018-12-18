# Use Cases

This document contains a list of use-cases we want to contain.

------
**Shortnames:**
	[PO]: PipeOperator
	[GN]: GraphNode
	[G] : Graph
	[GL]: GraphLearner

------
## Pipe Operators:

- **Structure:**
	- `pipeOpBranch`, `pipeOpSpread`         	| spread
	- `pipeOpChunk`								| spread
	- `pipeOpUnbranch` , `pipeOpGather`			| gather
	- `pipeOpFeatureUnion`						| gather
	- `pipeOpNULL`								| linear

- **Learner:**
	- `pipeOpLearner`							|			| task -> pred
	- `PipeOpLearnerCV`							|			| task -> pred
	- `pipeOpModelAverage`						|			| task -> pred

- **Preprocessing:**										| task -> task
	- `PipeOpPCA`
	- `PipeOpScale`
	- `PipeOpDownsample`

- **Target Operators:**										| task -> task
	- `PipeOpThreshold`	
	- `PipeOpTrafoY`
	- `PipeOpMultiClass2Binary`
	- `PipeOpsetTarget`


## Helper functions:

- `gunion(....)`
  Takes multiple graphs and puts them in one level (parallel to each other).
  Same as the output of a GraphOp that produces multiple outputs

- `greplicate(graph, k)`
  Takes a graph and copies it $k$ times and puts them in a list in
  the same way as `gunion(...)`.

- `>>%`

- `>=>%`


## Getter functions:

- `PO$id` Get the id of a PipeOp.


# FIXME: Unique PipeOp Params: Do this via **id.param**? 
# FIXME: Which hyperpars are exported by a pipeop?


## Use Cases 

------
### Usecase: Linear Pipeline

#### Usecase a): Linear Pipeline

*Scenario:* 
*We obtain a task (iris) and a learner(rpart) from mlr3.*
*Before we train the learner, we want to transform the data using PCA.*

We concatenate a `PipeOpPCA` and a `PipeOpLearner` using the `%>>%` (then) operator.
This internally does the following:
	- Wrap the [PO's] into **GraphNodes**.
	- Chains them together and returns a **Graph**.

```r
task = mlr3::mlr_tasks$get("iris")
lrn_rp = mlr3::mlr_learners$get("classif.rpart")
g = PipeOpPCA() %>>% PipeOpLearner(lrn_rp)
g$train(task)
g$predict(task)
```


##### [[pipeOpPCA]]

	- **train**: 
		- input: Task
		- does: Computes and stores rotation matrix into **.params** slot.
		- returns: Task
	- **params:**: rotation matrix
	- **predict**: 
		- input: Task
		- does: rotates input data using **.params** slot.
		- returns: Task

##### [[pipeOpLearner]]

	- **train**: 
		- input: Task
		- does: Calls the learner's`train()` method on it. Stores the model in **.params**.
		- returns: NULL
	- **params:**: trained model
	- **predict**:
		- input: Task
		- does: Calls the `predict()` method of the stored model.
		- returns: Prediction


#### Usecase b): Resample Linear Pipeline

*Scenario:* 
*We want to resample the Graph on different folds of the data.*

We create a `GraphLearner` from the Graph and use **mlr3's** resampling.

```r
lrn_g = GraphLearner$new(graph = g)
lrn$parvals = list(pca.center = TRUE, rpart.cp = 0.1) 
resampling = mlr_resamplings$get("holdout")
rr = resample(task, lrn_g, resampling)
```

Access results: 
```r
rr[1, "models"]$learner.model[["rpart"]]$learner.model
rr[1, "models"]$learner.model[["pca"]]$params
```

##### [[GraphLearner]]

	Wraps a Graph and allows it to be used like a learner.
	- **train**: 
		- input: Task
		- does: Calls the graph's `train()` method on it.
		- returns: NULL
	- **predict**:
		- input: Task
		- does: Calls the `predict()` method on the Graph.
		- returns: NULL

#### Usecase c): Tune Linear Pipeline

*Scenario:* 
*We want to tune the Graph.*

We use the `GraphLearner` and use **mlr3's** tuning.

```r
measures = mlr3::mlr_measures$mget("mmce")
param_set = paradox::ParamSet$new(
  params = list(
   ParamLgl$new("pca.center"),
   ParamDbl$new("rpart.cp", lower = 0.001, upper = 0.1)
)
ff = FitnessFunction$new(task, lrn_g, resampling, measures, param_set)
terminator = TerminatorEvaluations$new(10)
rs = TunerRandomSearch$new(ff, terminator)
tr = rs$tune()$tune_result()
```

------
### Usecase: Feature union

```r
op1 = PipeOpScaler$new()
op2a = PipeOpPCA$new()     # was machen wir hier mit den targets?
op2b = PipeOpNULL$new()
op3 = PipeOpFeatureUnion$new()
op4 = PipeOpLearner$new(learner = "classif.rpart")

op1 >> gunion(op2a, op2b) >> op3 >> op4
```

##### [[pipeOpScale]]

	- **train**: 
		- input: Task
		- does: Scales Task to mean 0 and sd 1. Stores mean and sd into **.params**
		- returns: Task
	- **params:**: mean, sd of all training data features.
	- **predict**: 
		- input: Task
		- does: scales input task using **.params** slot.
		- returns: Task

##### [[pipeOpNull]]

	- **train**: 
		- input: Anything
		- does: Nothing
		- returns: Input (Anything)
		- **params:**: Nothing
	- **predict**:
		- input: Anything
		- does: Nothing
		- returns: Input (Anything)

##### [[pipeOpFeatureUnion]]

	- **train**: 
		- input: List of Task's
		- does: Cbinds features from all tasks and one target col.
				(constraint: target cols and row ids all need to be the same,
			 	and same order).
		- returns: Task
	- **params:**: Nothing
	- **predict**:
		- input: List of Task's
		- does:  same as train (without targets, here we have to  do nothing).
		- returns: Task



### Usecase: Bagging (Downsampling, Modelling and Modelaveraging)

*Scenario:* 
*We want to do bagging (Train several models on subsamples of the data and 
average predictions.*

We use the `PipeOpDownSample` operator in conjunction with a `PipeOpLearner` to train a model. `greplicate()` let's us do the same operation multiple times.
Afterwards we average all predictions using `PipeOpModelAverage`

```r
op1 = PipeOpDownSample$new(rate = 0.6)
op2 = PipeOpLearner$new("classif.rpart")
op3 = PipeOpModelAverage$new()

greplicate(op1 %>>% op2, 30) %>>% op3
```

##### [[PipeOpDownsample]]

	- **train**: 
		- input: Task
		- does: Downsamples (samples rows with replacement) the input for a fraction of the original n.
		- returns: Task
	- **params:**: Nothing
	- **predict**: 
		- input: Task
		- does: Nothing
		- returns: Task

##### [[PipeOpModelAverage]]

	- **train**: 
		- input: Nothing
		- does: Nothing
		- returns: Nothing
	- **params:**: Nothing
	- **predict**: 
		- input: List of Predictions
		- does: Averages Predictions
		- returns: Prediction

# FIXME:
	Info: If our predictions are numeric, we simply average.
		  If our predictions are binary, we majority vote (?)
		  If our predictions are probabilities, we average (?)
		  If our predictions are multiclass, we (?).
		  Are there any other situations ? 

### Usecase: Stacking 

*Scenario:* 
*We want to do stacking (Train several models on the data and combine predictions).*

#### Usecase a): Stacking with simple Averaging

We use various `PipeOpLearner`'s' to train models. `gunion()` let's us put the learner's parallel to each other.
Afterwards we average all predictions using `PipeOpModelAverage`.

```r
op1 = PipeOpLearner$new("regr.rpart")
op2 = PipeOpLearner$new("regr.svm")
gunion(op1, op2) %>>% PipeOpModelAverage$new()
```

#### Usecase b): Stacking with SuperLearner


Instead of using `PipeOpModelAverage`, we combine predictions to a `PipeOpLearner`.
Instead of a `pipeOpLearner` we use a `PipeOpLearnerCV`, in order to avoid overfitting.

```r
# Superlearner: We instead use PipeOpLearnerCV 
op1 = PipeOpLearnerCV("regr.rpart")
op2 = PipeOpLearnerCV("regr.svm")
gunion(op1, op2) %>>% PipeOpFeatureUnion() %>>% PipeOpLearner("regr.lm")
```

##### [[PipeOpLearnerCV]]

	- **train**: 
		- input: Task
		- does: 1. Trains models an different folds of data. Predicts on holdout splits.
				2. Trains a model on full data, saves model to **.params**.
		- returns: Prediction
	- **params:**: trained model
	- **predict**: 
		- input: Task
		- does: Predict with model from .params
		- returns: Prediction

#### Usecase c): Stacking with SuperLearner and original data.

By adding a `pipeOpNull`, we add the original features to the SuperLearner.

```r
gunion(op1, op2, PipeOpNull) %>>% PipeOpFeatureUnion() %>>% PipeOpLearner("regr.lm")
```

### Usecase: Multiclass with Binary

*Scenario:* 
*We have a multiclass target, and want to predict each class in a binarized manner.*
*This occurs, for example if our model can only do binary classification.*

We use `PipeOpMultiClass2Binary` in order to split our task up into multiple binary tasks. 
Afterwards, we replicate our learner $k$ (where $k$ = number of classes - 1) times.
In order to aggregate the predictions for different classes, we use the `PipeOpModelAverage`.

```r
op1 = PipeOpMultiClass2Binary(codebook)
op2 = PipeOpLearner("classif.svm")

op1 %>>% greplicate(op2, k) %>>% PipeOpModelAverage$new()
# or:
op1 %>=>% greplicate(op2, k) %>>% PipeOpModelAverage$new()
```

##### [[PipeOpMultiClass2Binary]]

	- **train**: 
		- input: Task
		- does: Produces multiple tasks, where y is the binarized version of original y.
		- returns: List of Tasks
	- **params:**: Nothing
	- **predict**: 
		- input: Task
		- does: Copy Task k-times
		- returns: List of Tasks


### Usecase: Multiplexing of different Ops

*Scenario:* 
*We want our pipeline to branch out, either in one direction or the other.*
*This is usefull, for example when tuning over multiple learners.*

####  Usecase: Multiplexing different learners

We use the `pipeOpBranch` in order to have our data flow only to one of the following operators. 
Afterwards we collect the two streams using `PipeOpGather`.
We can now treat the pipeline like a linear pipeline.


```r
op1 = PipeOpLearner$new("regr.rpart")
op2 = PipeOpLearner$new("regr.svm")
g = pipeOpBranch$new(selected = 1) %>>% gunion(op1, op2) %>>% PipeOpGather(aggrFun = NULL)
```


##### [[PipeOpBranch]]

	- **train**: 
		- input: Anything
		- does: Creates list [NULL, NULL, X, NULL]; where X is at position **selected**.
		- returns: List of Task / NULLs
	- **params:**: Nothing
	- **predict**: Same as train.



####  Usecase: Multiplexing preprocessing steps

```r
op1 = PipeOpLearnerPCA$new()
op2 = PipeOpNULL$new()
op3 = PipeOpLearner$new("classif.rpart")

g = PipeOpBranch$new(selected = 1) %>>% gunion(op1, op2) %>>% PipeOpGather(aggrFun = NULL) %>>% op3
```

**FIXME:** Does every PipeOp have a default method when NULL is passed?


### Usecase: Chunk data into k parts, train on each, then model average

*Scenario:* 
*We want our pipeline to branch out, either in one direction or the other.*
*This is usefull, for example when tuning over multiple learners.*


We use the `pipeOpChunk` operator to partition the task into $k$ smaller tasks. 
Afterwards we train $k$ learners on each subtask.
Afterwards the predictions are averaged in order to get a single prediction.

```r
pipeOpChunk(k) %>>% greplicate(PipeOpLearner("classif.rpart"), 10) %>>% PipeOpModelAvg()
```


### Usecase: Thresholding

*Scenario:* 
*We want to obtain an optimal threshold in order to decide whether something is of class x or y.*

We use `PipeOpLearnerCV` to obtain cross-validated predictions. Afterwards we use `PipeOpThreshold()` to compute an
optimal threshold.

```r
op1 = PipeOpLearnerCV$new(""classif.rpart")
op2 = PipeOpThreshold$new(method, measure, ...)
g = op1 %>>% op2 
```

##### [[PipeOpThreshold]]

	- **train**: 
		- input: Prediction
		- does: Optimizes the threshold for a given performance metric.
		- returns: Prediction
	- **params:**: Optimal threshold
	- **predict**: 
		- input: Prediction
		- does: Binarizes the prediction using **.params**
		- returns: Prediction


------------------------------------ 
## Unknown territory, Here Be Dragons
------------------------------------ 


### Usecase: ThresholdingTrafo y (logarithm of y)

*Scenario:* 
*We want to transform our target variable, for example using a log-transform.*

We use the `PipeOpTrafoY` in order to log-transform the data.
We use another `PipeOpTrafoY` after the learner in order to re-transform our data onto the original scale. 

```r
top = PipeOpTrafoY(train = log, predict = identity)
retop = PipeOpTrafoY(train = identity, predict = exp)

g = top %>>% PipeOpLearner("classif.svm") %>>% retop
```

What happens: 
```
	- g$train(task) [[trafoY(log)) >>  train("classif.svm", task) %>>% identity]]
	- g$predict(task) [[identity >> predict(model, task) >> trafoPreds(exp)]]
```
 
##### [[PipeOpTrafoY]]

	- **train**: 
		- input: Task
		- does: Transforms target by fun.
		- returns: Task
	- **params:**:
	- **predict**: 
		- input: Prediction
		- does: Transform prediction by fun
		- returns: Prediction


FIXME:
	- Using the same operator twice would violate the acyclic property.
	- We can not tune over **par.vals** of TrafoY, as we have a hard time storing them.
	- User needs to ensure that trafos are correct

### Usecase: MultiOutput (1 Task, 3 Outputs, NoCV)


####  Usecase a): MultiOutput Parallel

*Scenario:* 
*We have three possible output variables we want to predict in parallel.*

We set different targets before training each learner using `PipeOpSetTarget`. 
Afterwards the different learners are collected with `PipeOpModelAverage`.

```r
g = gunion(
	PipeOpSetTarget("out1") %>>% PipeOpLearner("rpart", id = "r1"),
	PipeOpSetTarget("out2") %>>% PipeOpLearner("rpart", id = "r2"),
	PipeOpsetTarget("final_out") %>>% PipeOpLearner("rpart", id = "r3")
	) %>>% PipeOpModelAverage()
```

##### [[PipeOpSetTarget]]

	- **train**: 
		- input: Task
		- does: Set previous target col to hidden. Set new target variable.
		- returns: Task
	- **params:**:
	- **predict**:
		- input: Task
		- does: Nothing
		- returns: Task

####  Usecase b): MultiOutput Chained

*Scenario:* 
*We have three possible output variables available during training, but they will not be availalbe during test time.*
*We want to leverage info from out1 and out2 to improve prediction on final_out*

We obtain cross-validated predictions using `PipeOpLearnerCV` sequentially for each target and use them to train the sequential models.

```r
pnop = pipeOpNull()

g = PipeOpSetTarget("out1") %>>%
	gunion(PipeOpLearnerCV("rpart", id = "r1"), pnop) %>>%
	PipeOpFeatureUnion() %>>%
	PipeOpSetTarget("out2", id = "r2") %>>%
	gunion(PipeOpLearnerCV("rpart"), pnop) %>>%
	PipeOpLearnerCV() %>>%
	PipeOpsetTarget("final_out") %>>%
	PipeOpLearner("rpart", id = "r3") 
```




preproc operation: daten %>>% po --> preprozessierte Daten
trainierte preproc op auf neue daten: neuedaten %>>% (daten %>>% po) --> neuedaten insofern veraendert wie (daten%>>%po) es voreinstellt
training eines Modells: daten %>>% lrnpo --> modell trainiert auf daten
prediction: neuedaten %>>% (daten %>>% lrnpo) --> neuedaten predicted auf dem trainierten modell
splitOp: aufteilen in training und prediction data.
beide stroeme in einen knoten, und der macht dann beides?
trainierter knoten wird ausgespuckt und der predict-stream wird verwendet?
resampling mit  splitOp(k) %>>% rep(lrnpo, k) gibt direkt performances zurueck?
Alternativ splitOp(k) %>>% rep({lrnpo on training data, trained_learner on predicted data} %>>% measure performance, k)?
vll eher durch metagraphen
tuning: tunenode(<graph>), graph kriegt daten rein und gibt eine performance measure raus, und evtl auch ein predicted measure?
problem hier ist: die performance will man vll auf eine art messen (e.g. cv), die tatsaechliche Anwendung dann auf den ganzen Datensatz, aber ein grossteil der operation ist schon der gleiche.
zeilennummern will man schon irgendwie behalten
send parameter confs through the graph




# graph definition:
# op1, op2, pma exist and are supposed to be connected by
# op1 
#          > pma
# op2 
GraphDef({
	op1 %>>% pma
	op2 %>>% pma
	# all that have no output automatically output to SINK
	# all that have no input automatically input from SOURCE
	# but SOURCE and SINK can be given explicitly
	SOURCE %>>% pma
})

