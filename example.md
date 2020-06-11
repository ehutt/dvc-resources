# Using DVC: End-to-End Example

## Situation: 
We have a large audio dataset of support calls (~70,00). We would like to take a small sample (500 calls) and
transcribe it using one of our STT APIs (an ASR model). We want to then generate human-readable transcripts 
which have the utterances labeled by speaker - either "Agent" or "Caller."

## Overview of Steps: 
1. Add raw audio dataset to DVC data registry and generate some statistics about it. 
2. Get a 500 call sample of the dataset according to some set of constraints, generate statistics for the sample, 
and save to a separate directory. 
3. Import the sample audio to a dvc project for performing ASR. 
4. Process the audio - split into channels, upsample if necessary, ensure the correct encoding, etc. 
5. Run the ASR on the audio data in parallel to get raw transcripts for the audio. 
6. Process the results - take the raw ASR output, merge the channels, and label the speakers 
using simple pattern matching. 

### 1. Add raw audio dataset to DVC data registry 
Assume we downloaded the raw audio from "wherever". We want to copy it to our local dvc-data-registry repo. 
```
cp ~/Downloads/raw_audio ~/github_projects/wccai-dvc-data-registry/raw/audio/calls/example/dataset
cd ~/github_projects/wccai-dvc-data-registry
dvc add raw/audio/calls/example/dataset
```
Whenever we add a new dataset to the registry, we also want to save some metadata about it. 
Since each dataset is slightly different, I usually have to write a new script for extracting this information from 
a dataset. 
``` 
dvc run -f dvc_files/01_example_file_metadata.dvc \
    -d raw/audio/calls/example/dataset \
    -d code/process_example.py \
    -o raw/audio/calls/example/metadata \
    python code/process_example.py --input_path raw/audio/calls/example/dataset \
    --output_path raw/audio/calls/example/metadata
```
The `process_example_audio.py` script will look at each individual wav file in the input path and output a json 
with information about it, for example audio duration and any other metadata that might be available. 

**Note:** in the above `dvc run` command, I specified that the `metadata` directory is an output of the script that 
should be tracked. Notice that this directory doesn't yet exist - it is created inside the python script. This is because 
dvc requires that any declared outputs (`-o, --outputs`) are created strictly within the scope of the stage. 

**Note:** I have been storing all my dvc stage files in a folder called `dvc_files` for the sake of cleanliness,
but starting with the new release of Version 1.0, dvc stores all stage information in a single file, so this won't be necessary. 

Once we have extracted and stored metadata for each file, we also want to generate some information about the dataset in aggregate. 
For datatypes we work with frequently, we have defined schemas which should be used to generate this metadata, for example:
* Audio source
* Original raw or modified from 
* Company
* Date acquired or modified
* Data type (e.g. call)
* Dataset size (e.g. in GB)
* Number of files
* Format (e.g. wav)
* Sample rate (e.g. 8000)
* Stereo or mono 
* Encoding (e.g. Linear 16 PCM)
* Total number of hours (Hrs:Min:Sec)
* Mean, median, min and max Duration
* Public vs. Private 
* Contains PII

While most of fields must be filled out manually, some can be generated automatically from the file metadata we generated earlier. 
I am still trying to find a way to make this data cataloging process more automatic and easy. 

I store this information in a json file called `info.json` in the dataset directory and DO NOT use dvc to track this - it gets 
versioned with git so that anyone on the team can see what's the dataset is all about. 

### 2. Get a 500 call sample of the dataset

Now that we have our dataset catalogued and versioned with dvc, we can generate the sample we want to transcribe. For this, 
I have a script called `sample.py` which takes as input a `params.yaml` file which contains the following constraints for the sample: 
``` 
audio_sample:
    sample_size: 500
    min_dur: 60 
    max_dur: 1800
```
To run on our dataset and store the sample in a separate directory, all tracked by dvc, run: 
``` 
dvc run -f dvc_files/02_sample_example.dvc \
    -d raw/audio/calls/example/dataset \
    -d code/sample.py \
    -p audio_sample \
    -o audio/calls/example/sample \
    python code/sample.py --params params.yaml --input raw/audio/calls/example/dataset \
    --output audio/calls/example/sample
```
**Note:** again, the `sample` directory gets created at runtime. 
**Note:** The `-p` flag in `dvc run` indicates the presence of a special dependency, by default stored in a file called `params.yaml`. 
If any of the fields under the `audio_sample` section of the yaml file changes, then dvc will detect this and any calls to `dvc repro` will 
rerun with the updated parameters.

After getting the sample, we rerun the cataloging steps from before to get metadata/stats for the sample dataset. Finally, call
`dvc push` to push all the new data to our remote - an s3 bucket that we configured upon initialization of the data registry.

``` 
git add . && git commit -m "add example data" 
dvc push
```

### 3. Import the sample audio to a dvc project for performing ASR. 

Now that all our audio data is tracked and pushed to the cloud, we can use it in any of our other projects. For this example, 
we move to our git repo for performing transcription and use `dvc import` to pull the sample data into that workspace. 
``` 
cd audio/raw
dvc import git@sqbu-github.cisco.com:CCBU-CCOne/wccai-dvc-data-registry.git \
    audio/calls/example/sample
```
Using `dvc import` rather than `dvc get` will preserve the string of dependencies which produced this version of the data. 

### 4. Process the audio
Once again, we need to process this data so it is in the correct format for the ASR model. For this we 
use a bash script `preprocess.sh` which asserts the audio is in the correct sample rate/encoding, splits the 
stereo audio into left and right channels, and stores in 3 separate directories (for the sake of parallel processing).

``` 
dvc run -f dvc_files/01_preprocess_example.dvc \
    -d audio/raw/sample 
    -d code/preprocess.sh \
    -o audio/clean \
    bash preprocess.sh audio/raw/sample audio/clean
```

### 5. Run the ASR
With our wav files split into left and right channels and stored in 3 directories, it is easy to 
call the STT API in parallel over the 3 sets of data. We use a bash script, which internally calls 
a python script, to perform the ASR. Again, the script reads from a `params.yaml` file where we define 
the parameters for a run.


params.yaml
``` 
api:
  token_generator: "code/api/AccessToken.sh"
  token_file: "api_token.json"
  token_refresh: 43199
  host: "asr-proxy.uscentral1-0.cint.vcra.co"
  port: 443
  tracking_id: "CiscoIt_sample"
audio:
  sample_rate: 8000
  encoding: 1

``` 

Run:
``` 
dvc run -dvc_files/02_transcribe_example.dvc \
    -d audio/clean \
    -d transcribe_parallel.sh \
    -d code/transcribe.py \
    -p api, audio \
    -o results/raw \
    bash transcribe_parallel.sh audio/clean results/raw 
```

### 6. Process the results
Once we get the raw results from the ASR model, we need to do some more processing. Namely, we need to merge 
the transcripts corresponding to the left and right audio channels, generate a human-readable transcript from that, 
and then try to identify which of the two speakers is the agent and which is the caller. We do this with 
a script called `postprocess.sh` which merges the transcripts, generates metadata for them, and internally 
calls a python script `label_speakers.py` which tries to match transcripts to a set of regular expressions. 

For example, the regular expressions could be: 
``` 
agent_expressions = [
        r'how (may|can) I (help|assist)',
        r'(help|assist) you'
    ]
```
We want to measure how many of the transcripts are getting labeled "agent" and "caller" so we output that number 
as a metric as well. 
Run: 
``` 
dvc run -f dvc_files/03_postprocess_example.dvc \
    -d results/raw \
    -d code/postprocess.sh \
    -d code/label_speakers.py \
    -o results/readable/transcripts \
    -o results/readable/metadata \
    -m labeled.json \
    bash code/postprocess.sh results/raw results/readable

git add . && git commit -m "run stt pipeline for example data" 
git tag -a baseline -m "baseline stt with agent labeling" 
git push origin baseline
```

### 7. BONUS - iterate and improve 
Now that we have some human-readable, speaker-separated transcripts, we skim some of them and notice a few things. 
For example, we see that our regular expression matching only managed to label a few calls with "Agent" and "Caller" but
but we see that we could expand our regular expression list to cover some more bases. In many of the transcripts, 
the agent (presumably) says "thank you for calling so-and-so" - let's add this to our list and see if we can label more 
transcripts. To do this, we simply modify the `label_speakers.py` script by adding to our list of expressions from above, which 
should now look like this: 

``` 
agent_expressions = [
        r'thank you for (calling|contacting)',
        r'how (may|can) I (help|assist)',
        r'(help|assist) you'
    ]
```

Simply call `dvc repro` and the postprocessing step will rerun, updating 
the dvc-tracked outputs to reflect the changes we made. 

``` 
dvc repro dvc_files/03_postprocess_example.dvc 
git add . && git commit -m "rerun agent labeling with new expressions" 
git tag -a added_expression -m "added thank for calling to agent labeling step" 
git push origin added_expression
```

To see if adding that expression helped to label more of the transcripts, compare the metrics by doing:

``` 
git checkout master 
dvc metrics diff
```

This command will show us that we were able to label 35 more of the transcripts with the new expression! 

### 8. Save and Catalog the results 

We want our transcripts to be available for others on the team to use, so let's add them to the data registry. 
One way to do this is to simply copy the data to your data registry repo, `dvc add` and then `dvc push` them.
But this will lose the dependencies. 

An alternative is to create a remote for this transcription project, push to that, then import from the data registry. 
For example: 

``` 
dvc remote add -d transcripts s3://datasets/transcripts
dvc remote modify datasets profile dvc
git add . && git commit -m "setup transcripts remote" && git push
dvc push dvc_files/03_postprocess_example.dvc 
```

**Note:** this `dvc push` command will only push the OUTPUTS of the specified .dvc stage. To push all artifacts and dependencies,
you can use the `--with-deps` flag. Here, we are only interested in the output transcripts. 

Now, go back to the data registry repo and run: 
``` 
mkdir text/calls/example/sample && cd text/calls/example/sample
dvc import git@sqbu-github.cisco.com:CCBU-CCOne/wccai-nutcracker-transcription.git \
    results/readable/transcripts
dvc import git@sqbu-github.cisco.com:CCBU-CCOne/wccai-nutcracker-transcription.git \
    results/readable/metadata 
dvc push 
```
**Note:** make sure you also get the `info.json` file if you made one for the dataset. 

Now you and your team can use this transcript dataset in any of your other projects. Simply call `dvc update` + the name 
of the .dvc file to pull in any updates to the dataset which may have occurred upstream. 

