# dvc-resources

# Data Version Control (DVC) 

In this tutorial, we cover: 
1. DVC basics 
3. How to build and use a data registry 
4. How to work on an existing DVC project 
5. How to conduct ML experiments using DVC 

This tutorial assumes familiarity with dvc's basic usage, so if you're not there yet I highly recommend you check out 
their [documentation](https://dvc.org/doc/home) and their [introductory tutorial.](https://dvc.org/doc/tutorials/get-started/agenda) 
No need for me to paraphrase - they have done a phenomenal job with their documentation. 


## The Basics 

DVC is an open-source development tool designed with data science and machine learning practitioners in mind. Unlike a traditional software 
development workflow, machine learning often involves a multi-step cycle of data processing, compute-intensive model training, 
hyperparameter tuning, and performance evaluation. It is highly iterative and experimental in nature, not to mention data hungry and computationally demanding. 
DVC is specifically designed to fill the gap between existing software development tools and the unique needs of data scientists. More than simply 
providing data version control, with DVC you can efficiently build, compare, reproduce, and share all components or versions of a machine learning pipeline. 

I could go on and on about how game-changing DVC has been, so here are just a few of my favorite features: 
* the git-like command line interface is easy to learn and use
* extends the benefits of git-based version control to large files like data and models
* integration with cloud storage systems (we use AWS s3) means data are both safe and accessible to everyone on the team
* efficient data and model management eliminates the need for repeated steps and makes them easy to share
* ability to conduct ML experiments such that results are completely reproducible and metrics are easily compared across different 
versions of a pipeline via git tags and branches
* all-in-one solution - in searching for ways to optimize my ML workflow, I looked at other options like MLFlow and AWS Sagemaker, 
but none of them were as intuitive or as complete as DVC is. 

DVC Resources: 
* [Github](https://github.com/iterative/dvc) 
* [documentation]
* [Introductory Tutorial](https://dvc.org/doc/tutorials/get-started/agenda)


## How Build and Use a Data Registry 

### What is a data registry? 
A data registry is simply a DVC project dedicated to the storage and tracking of data and/or models. It provides an interface 
to your remote storage system of choice, through which you can use and update large data files in a reproducible manner. 
There are several notable advantages of using such a system, best explained by the dvc makers themselves:

![alt text](https://github.com/ehutt/dvc-resources/images/dataregistry.png)

I will be using AWS S3 for the remote storage in this tutorial since that is what we use on my team, but the principles are the same
whether you are using local storage, GCP, SSH, etc. For a complete list of supported remote storage types, see [here](https://dvc.org/doc/command-reference/remote/add)

### Setup 
1. Set up remote storage 
    
      Suppose we have an s3 bucket called `datasets` which we use to store all our data and we have our aws access credentials stored in a profile called `dvc`
2. Initialize your dvc project and remote
    
    In a fresh git repo, initialize dvc, add the s3 bucket as a remote and modify the profile to ensure access, then commit
    the changes. The `-d` flag indicates that this will be the dvc project's default remote. 
    
    ``` 
    dvc init
    dvc remote add -d datasets s3://datasets/dvc-data-registry
    dvc remote modify datasets profile dvc
    git add . && git commit -m "setup dvc remote" && git push
    ```


### Adding Data to the Registry 

I work with a combination of audio and text data. The workflow usually involves first
transcribing audio using an ASR model and then performing some kind of NLP on the resulting transcripts. 
I like to keep raw and processed data separate, and then organize the subdirectories according to type. So for example, my data registry might 
have the following structure: 

```
sample-data-registry
├── raw
│   ├── audio
│   ├── text
├── text
│   ├── transcripts
```

Now I want to add some raw audio data to my dvc data registry, a file called `recording.wav` that I have stored locally. 

1. Copy raw data to appropriate `raw` directory and track with dvc by calling `dvc add` 
    ``` 
    mkdir raw/audio
    cp ~/Downloads/recording.wav raw/audio/
    dvc add raw/audio/recording.wav
    git add . && git commit -m "add raw recording data" && git push
    ```

2. Push data to remote storage 

    ``` 
    dvc push 
    git add . && git commit -m "push recording data to s3" && git push 
    ```



### Using Data from the Registry 

#### Import data to your local workspace and use it in a project

Now that my raw audio file, `recordings.wav` is catalogued in my data registry and stored in s3, I want to transcribe it. 
I already have a dvc project in a repo for doing transcription with an ASR model. First I have to import the raw data into 
the transcription project repo. 

1. First let's get a list of what is available in the data registry (`-R` flag searches recursively to find all files in the repo): 
    ``` 
    dvc list -R https://github.com/ehutt/sample-data-registry
    
    ``` 
    ```
    README.md
    raw/audio/.gitignore
    raw/audio/recording.wav
    raw/audio/recording.wav.dvc
    ```
    We can see our recording there, now let's import it. We always use `dvc import` as opposed to `dvc get` since the former
    maintains any dependencies on the data source (i.e. the registry repo) while the latter is a simple download or copy. 
    
2. Import data to your local workspace (must be an initialized dvc project) by specifying the registry repo and the 
path to the desired data.

    ``` 
    dvc import https://github.com/ehutt/sample-data-registry \
    raw/audio/recording.wav
    ```

### Create a pipeline stage

Now that I have my imported audio data, now I can run my ASR model to get a transcription. I use a `dvc run` command to track
the transcription stage, which will keep track of which data, model, and parameters were used to generate the transcription. The `-d` flag
indicates a dependency (here, dependencies are the raw audio and the model code) and `-o` indicates an output (here, the transcript).
    ``` 
    dvc run -f transcribe.dvc \
    -d recording.wav \
    -d transcribe.py \
    -o transcript.txt \
    python transcribe.py recording.wav 
    ```
Running this stage generates a transcript file, which I now want to add to my data registry. To do so, simply copy the file 
to the registry repo and called `dvc add` again. Now the data registry looks something like this: 

    ```
    data-registry
    ├── raw
    │   ├── audio
    │       ├──recording.wav
    ├── text
    │   ├── transcripts
    │       ├──recording.txt     
    ```

Other tips and tricks for using a data registry 

* To make sure you have the latest version of an imported dataset, run `dvc update` to pull any changes from 
* In the situation where you want to modify a raw dataset and store the result in your data registry, it is also possible 
to perform the steps from within the registry repo itself. In mine, I have a folder to store code so that when I do any data processing 
(e.g. take some raw text and remove html artifacts), I can maintain the dependencies using `dvc run`


* I like to store all my dvc stage files in a dedicated folder called `dvc_files` to avoid cluttering the project repo. 
