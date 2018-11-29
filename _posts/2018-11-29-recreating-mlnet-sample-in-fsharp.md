---
layout: single
title: Recreating the ML.NET Getting Started Tutorial in F#
date: 2018-11-29
categories: [FSharp]
tags: [ML.NET]
comments: true
share: true
---

Microsoft have two tutorial sites for ML.NET. One at [dotnet.microsoft.com](https://dotnet.microsoft.com/learn/machinelearning-ai/ml-dotnet-get-started-tutorial) and one at [docs.microsoft.com](https://docs.microsoft.com/en-us/dotnet/machine-learning/tutorials/).

Each of the tutorials in the second link have the F# equivalent on [GitHub](https://github.com/dotnet/machinelearning-samples/tree/master/samples/fsharp). However, GitHub does not have the F# sample for the [dotnet.microsoft.com](https://dotnet.microsoft.com/learn/machinelearning-ai/ml-dotnet-get-started-tutorial) tutorial. You can find implementations of it on Google but most of them use the legacy `LearningPipeline` and not the new `MLContext` used in the tutorial.

### Creating the project

To begin, call the following commmands to create your project.

```cmd
dotnet new console -lang F# -o myApp
cd myApp
```

After this, you can mostly follow the [tutorial](https://github.com/dotnet/machinelearning-samples/tree/master/samples/fsharp) to get the data set and add the ML.NET package.

### The Code

The F# equivalent of the tutorial code is mostly straightforward and is displayed below.

```fsharp
open System
open Microsoft.ML
open Microsoft.ML.Runtime.Api
open Microsoft.ML.Runtime.Data
open Microsoft.ML.Core.Data

// STEP 1: Define your data structures
// IrisData is used to provide training data, and as
// input for prediction operations
// - First 4 properties are inputs/features used to predict the label
// - Label is what you are predicting, and is only set when training
[<CLIMutable>]
type IrisData = {
        SepalLength : float32
        SepalWidth : float32
        PetalLength : float32
        PetalWidth : float32
        Label : string
    }

[<CLIMutable>]
// IrisPrediction is the result returned from prediction operations
type IrisPrediction = {
        [<ColumnName("PredictedLabel")>]
        PredictedLabel : string
    }


[<EntryPoint>]
let main argv =
    // STEP 2: Create a ML.NET environment  
    let mlContext = new MLContext()

    // If working in Visual Studio, make sure the 'Copy to Output Directory'
    // property of iris-data.txt is set to 'Copy always'
    let dataPath = "iris-data.txt";
    let reader =
        mlContext.Data.TextReader(
            TextLoader.Arguments(
                Separator = ",",
                HasHeader = true,
                Column =
                    [|
                        TextLoader.Column("SepalLength", Nullable DataKind.R4, 0)
                        TextLoader.Column("SepalWidth", Nullable DataKind.R4, 1)
                        TextLoader.Column("PetalLength", Nullable DataKind.R4, 2)
                        TextLoader.Column("PetalWidth", Nullable DataKind.R4, 3)
                        TextLoader.Column("Label", Nullable DataKind.Text, 4)
                    |]
            )
        )
    let trainingDataView = reader.Read(MultiFileSource(dataPath))

    // Helper functions to help with creating the pipeline
    let append (estimator : IEstimator<'a>) (pipeline : IEstimator<'b>)  =
        match pipeline with
        | :? IEstimator<ITransformer> as p ->
            p.Append estimator
        | _ -> failwith "The pipeline has to be an instance of IEstimator<ITransformer>."

    let downcastPipeline (pipeline : IEstimator<'a>) =
        match pipeline with
        | :? IEstimator<ITransformer> as p -> p
        | _ -> failwith "The pipeline has to be an instance of IEstimator<ITransformer>."

    // STEP 3: Transform your data and add a learner
    // Assign numeric values to text in the "Label" column, because only
    // numbers can be processed during model training.
    // Add a learning algorithm to the pipeline. e.g.(What type of iris is this?)
    // Convert the Label back into original text (after converting to number in step 3)
    let pipeline =
        mlContext.Transforms.Categorical.MapValueToKey("Label")
        |> append(mlContext.Transforms.Concatenate("Features", "SepalLength", "SepalWidth", "PetalLength", "PetalWidth"))
        |> append(mlContext.MulticlassClassification.Trainers.StochasticDualCoordinateAscent(label="Label", features="Features"))
        |> append(mlContext.Transforms.Conversion.MapKeyToValue("PredictedLabel"))
        |> downcastPipeline

    let model = pipeline.Fit(trainingDataView)

    let prediction =
        model.MakePredictionFunction<IrisData, IrisPrediction>(mlContext).Predict(
            {
                SepalLength = 3.3f
                SepalWidth = 1.6f
                PetalLength = 0.2f
                PetalWidth = 5.1f
                Label = ""
            }
        )

    printfn "Predicted flower type is: %s" prediction.PredictedLabel
    0 // return an integer exit code
```

One big difference from the C# code is in the way the pipeline is created. Due to how F# deal with type constraint, it is necessary to do a type check to call the `Append` function and to downcast it. In the code above, two functions is created to make the process easier.
