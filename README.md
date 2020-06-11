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