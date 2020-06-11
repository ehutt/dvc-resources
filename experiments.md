## How to Conduct Machine Learning Experiments Using DVC 

Let's run a toy experiment to compare the performance of two question-answering models. Suppose we have a labeled dataset
a la [SQuad2.0](https://rajpurkar.github.io/SQuAD-explorer/), with some paragraphs and corresponding question-answer pairs. 
We have two pre-trained models from [Hugging Face](https://huggingface.co/transformers/) transformers, `distilbert-base-uncased` and `albert-base-v2-uncased`, 
which have been fine-tuned for a question answering task and we want to compare them across two metrics: F1 and inference speed. 
In our DVC project, we have an evaluation script that outputs these scores and a config file 
that specifies parameters for the evaluation, including which model should be used. 

First, make sure you are somewhat familiar with the following cool DVC features: 
* [`dvc params`](https://dvc.org/doc/command-reference/params) for assigning parameters to pipeline stages
* [`dvc metrics`](https://dvc.org/doc/command-reference/metrics) for specifying outputs that you want to track as metrics 

Steps: 
1. First, let's make a new branch off of master for our experiment 

    `git checkout -b qa-experiment`

2. Let's make a directory to store the results of our experiment and the annotated data. 
    
    ``` 
    mkdir results 
    mkdir data 
    ```
    
3. Import the annotated data from our handy data registry. 

    ``` 
    dvc import https://github.com/ehutt/sample-data-registry \
    text/annotations/qa.json 
    mv qa.json data
    ```
3. Import the models from our handy model registry. 

    ``` 
    dvc import https://github.com/ehutt/sample-model-registry \
    question-answering/
    mv question-answering/ models/
    ```
4. Take a look at the config file, `params.yaml`. The evaluation script will read from this file to load the specified 
model checkpoints and other arguments for running model inference. 
    ``` 
    model: 
        path: models/distilbert
    args:
        max_answer_length: 30
        topk: 1
    ```
5. Create a DVC pipeline stage for performing evaluation, using the `-p` flag to specify which parameters are used 
and the `-M` to specify that the output is a metric. 
    ```
    eval_data=data/qa.json
   
    dvc run -f evaluate.dvc \
    -d evaluate.py \
    -d $eval_data \
    -p model, args \
    -M results/scores.json \
    python evaluate.py $eval_data results
   ```
4. Commit and tag the results, adding a short description with the model version and task mentioned.

    ``` 
    git add . && git commit -m "evaluate distilbert qa"
    git tag -a distilbert -m "distilbert_qa" 
    git push origin distilbert 
    ```
   
5. Edit `params.yaml` to use a different model - albert instead of distilbert this time. 

    ``` 
    model: 
        path: models/albert
    ```
   
6. Reproduce the run - DVC automatically recognizes that one of the dependencies (`params.yaml`) 
has changed and will re-run the stage, implementing the changes. 

    `dvc repro evaluate.dvc`

7. Commit and tag the results 

    ``` 
    git add . && git commit -m "evaluate albert qa"
    git tag -a albert -m "albert_qa" 
    git push origin albert
    ```
8. Now compare the results! Switch back to the master branch. DVC will search the git branches 
for those metric files we specified and display the results. 

    ``` 
    git checkout master 
    dvc metrics show -T
    ```
   output:
    ``` 
    distilbert:
        results/scores.json: {"f1": 67.87348075352251, "avg_inference_time":  0.38483647589987896 }
    albert:  
        results/scores.json: {"f1": 81.7724930324639, "avg_inference_time":  0.8740714052890209 }
    ```
    The use of DVC metrics makes it easy to compare models in this experiment. From these results, we can see that
    the albert model is more accurate, but the distilbert model is twice as fast. Good to know!    
   
9. Once you are done with your experiment, merge the branch of the best version back to master.
Don't delete the experiment branch - it acts as a record of the experiment you ran. 

### Experiment Tips and Tricks 
* Create a different branch for each experiment you intend to run, then use git tags to label each run of your experiment. 
As always, only modify one independent variable per experiment. 
* Use a single `params.yaml` file to configure your project, this way you only need to edit one file for each 
experiment run. 
