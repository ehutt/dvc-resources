# Data Version Control (DVC) 

DVC is an open-source development tool designed with data science and machine learning practitioners in mind. Unlike a traditional software 
development workflow, machine learning often involves a multi-step cycle of data processing, compute-intensive model training, 
hyperparameter tuning, and performance evaluation. It is highly iterative and experimental in nature, not to mention data hungry and computationally demanding. 
DVC is specifically designed to fill the gap between existing software development tools and the unique needs of data scientists. More than simply 
providing data version control, DVC allows you to efficiently build, compare, reproduce, and share all components or versions of a machine learning pipeline. 

I could go on and on about how game-changing DVC has been, so here are just a few of my favorite features: 
* the git-like command line interface is easy to learn and use
* extends the benefits of git-based version control to large files like data and models
* integration with cloud storage systems (we use AWS s3) means data are both safe and accessible to everyone on the team
* efficient data and model management eliminates the need for repeated steps and makes them easy to share
* ability to conduct ML experiments such that results are completely reproducible and metrics are easily compared across different 
versions of a pipeline via git tags and branches
* all-in-one solution - in searching for ways to optimize my workflow, I looked at other options like MLFlow, Tensorboard and AWS Sagemaker, 
but none of them were as intuitive or as complete as DVC. 

In this repo, I provide examples and in-depth commentary on the following concepts:

3. How to build and use a data registry 
5. How to conduct  experiments using DVC 
3. How to Use DVC End-to-End 

These examples assume familiarity with DVC's basic usage, so if you're not there yet I highly recommend you check out 
their [documentation](https://dvc.org/doc/home) and this [introductory tutorial.](https://dvc.org/doc/tutorials/get-started/agenda) 
No need for me to paraphrase - they have done a phenomenal job explaining how it works. 


## Conclusion
DVC has totally transformed the way I manage data science projects, somehow without making me change my entire workflow. I love that I did 
not have to learn a bunch of fancy new commands, mess around with superfluous GUIs, or struggle with managing external servers. 

These two core features of DVC - the data registry and experiment tracking - have been especially useful to me, but there were not many examples of how to 
use these tools in practice. I hope this tutorial helps you to better understand the power of DVC and to see how truly simple it can be. 

# FAQs 

**Q: Is my data secure?**

A: Yes! Your data is as secure as the platform you use to store it. DVC collects anonymized usage statistics but never sees your data - details [here](https://dvc.org/doc/user-guide/analytics). 
It lives only in your local cache and your remote storage.
Note that there are some special policies if you are using Google APIs - details [here](https://dvc.org/doc/user-guide/privacy). 

**Q:  The value of DVC seems to be very dependent on the data scientist or developer being committed to integrating it into their workflow. Do you have any tips for integrating the DVC workflow with the git workflow?**

A: Yes, it is true that success is dependent on the data scientist actually integrating the tool. But, I think that of the options available, DVC is probably the easiest to learn and use. 
I personally think Git is an amazing tool and I encourage anyone who writes code to use it for version control. 
As someone who already uses and likes git, learning to incorporate dvc into my workflow was very straightforward. If you would like to automate the process further, 
you could add a hook to check for uncommited changes. Also, if you plan to use a dvc-controlled data registry, this helps enforce DVC usage within a team since it forces everyone to at least learn how to push and pull data via DVC.

**Q: One of the challenges of working with Git in a team environment is trying to resolve merge conflicts. Do you experience the same sort of challenges with DVC?**

A: DVC will never overwrite data nor will it store redundant data. Because DVC stores everything in a content-addressable way, there are no conflicts. 
If you have two data scientists pushing different versions of the same dataset to the cloud, then DVC will recognize both versions and associate them with their respective git commits. 

**Q: How does DVC compare to Airflow?**

A: DVC addresses Airflow as a related technology [here](https://dvc.org/doc/understanding-dvc/related-technologies). 

**Q: How does DVC compare to MLFlow?**

A: I think there are a few key differences between MLFlow and DVC, namely:

* DVC is command line, MLFlow imported and used inside code
* DVC optimized for large file storage, MLFlow doesn’t seem to be
* MLFlow has a GUI, DVC does not.

Which one you use will depend on your use case. If your main concern is data versioning, I think DVC is the way to go. If you are more interested in running experiments/training models, then MLFlow seems well-suited to handle large hyperparameter searches. 
For my current project, the search space of possible experiments is smaller so for the sake of simplicity I will probably track them with DVC, especially since the code is already written and I don’t want to add MLFlow to all the components of the pipeline.

**Q: When you call `dvc push`, does that push the entire dataset to remote storage each time or simply a digest of the changes from the original dataset to the current commit?**

A: When you call `dvc push`, DVC compares the contents of your local cache with those of your remote storage. 
It will only copy over data that has changed or is new. 
