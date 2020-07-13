
# Multi-person Real-time Action Recognition Based-on Human Skeleton

![](https://github.com/felixchenfy/Data-Storage/raw/master/EECS-433-Pattern-Recognition/recog_actions.gif)

![](https://github.com/felixchenfy/Data-Storage/raw/master/EECS-433-Pattern-Recognition/recog_actions2.gif)


**Highlights**: 
4 actions; single people (<=5); Real-time and multi-frame based recognition algorithm. 

**Contents:**
- [1. Algorithm](#1-algorithm)
- [2. Install Dependency (OpenPose)](#2-install-dependency--openpose-)
- [3. Program structure](#3-program-structure)
  * [Diagram](#diagram)
  * [Main scripts](#main-scripts)
- [4. How to run: Inference](#4-how-to-run--inference)
  * [Introduction](#introduction)
  * [Test on video file](#test-on-video-file)
  * [Test on a folder of images](#test-on-a-folder-of-images)
  * [Test on web camera](#test-on-web-camera)
- [5. Training data](#5-training-data)
  * [Download my data](#download-my-data)
  * [Data format](#data-format)
  * [Classes](#classes)
- [6. How to run: Training](#6-how-to-run--training)
- [7. Result and Performance](#7-result-and-performance)



# 1. Algorithm


we collected videos of 4 Types of actions: `['stand', 'walk', 'sit', 'meal']`. 
We used the following dataset for training:  
*  CMU Panoptic Dataset (http://domedb.perception.cs.cmu.edu/)  
*  CAD-120 (http://pr.cs.cornell.edu/humanactivities/data.php) (3D point extracted using Lifting from the Deep: https://github.com/DenisTome/Lifting-from-the-Deep-release)   
*  Two videos (https://www.youtube.com/watch?v=rXjx5Jplfz8, https://www.youtube.com/watch?v=Jn52_g7CKx0) (3D point extracted using Lifting from the Deep: https://github.com/DenisTome/Lifting-from-the-Deep-release)

The workflow of the algorithm is:
*  Get the joints' positions (this can be done with Lifting from the Deep: https://github.com/DenisTome/Lifting-from-the-Deep-release).  
*  Track person. Euclidean distance between the joints of two skeletons is used for matching two skeletons. 
See `class Tracker` in [lib_tracker.py](utils/lib_tracker.py)
*  Fill in a person's missing joints by these joints' relative pos in previous frame.  See `class FeatureGenerator` in [lib_feature_proc.py](utils/lib_feature_proc.py). So does the following.
*  Use a window size of 0.5s (5 frames) to extract features.    
*  Extract features of (1) body velocity and (2) normalized joint positions and (3) joint velocities.
*  Apply PCA to reduce feature dimension to 80.  Classify by DNN of 3 layers of 50x50x50 (or switching to other classifiers in one line). See `class ClassifierOfflineTrain` in [lib_classifier.py](utils/lib_classifier.py)
*  Mean filtering the prediction scores between 2 frames. Add label above the person if the score is larger than 0.8. See `class ClassifierOnlineTest` in [lib_classifier.py](utils/lib_classifier.py)

For more details about how the features are extracted, please see my [report](https://github.com/felixchenfy/Data-Storage/blob/master/EECS-433-Pattern-Recognition/FeiyuChen_Report_EECS433.pdf).



# 2. Install Dependency (OpenPose)

First, Python >= 3.6.

I used the OpenPose from this Github: [tf-pose-estimation](https://github.com/ildoonet/tf-pose-estimation). First download it:

```
export MyRoot=$PWD
cd src/githubs  
git clone https://github.com/ildoonet/tf-pose-estimation  
```

Follow its tutorial [here](https://github.com/ildoonet/tf-pose-estimation#install-1) to download the "cmu" model. As for the "mobilenet_thin", it's already inside the folder.  

```
$ cd tf-pose-estimation/models/graph/cmu  
$ bash download.sh  
```

Then install dependencies. I listed my installation steps as bellow:
```
conda create -n tf tensorflow-gpu
conda activate tf

cd $MyRoot
pip install -r requirements.txt
pip install jupyter tqdm
pip install tensorflow-gpu==1.13.1
sudo apt install swig
pip install "git+https://github.com/philferriere/cocoapi.git#egg=pycocotools&subdirectory=PythonAPI"

cd $MyRoot/src/githubs/tf-pose-estimation/tf_pose/pafprocess
swig -python -c++ pafprocess.i && python3 setup.py build_ext --inplace
```

Make sure you can successfully run its demo examples:
```
cd $MyRoot/src/githubs/tf-pose-estimation
python run.py --model=mobilenet_thin --resize=432x368 --image=./images/p1.jpg
```

# 3. Program structure

## Diagram

![](doc/system_diagram.png)

Trouble shooting:
  * How to change features?

    In [utils/lib_feature_proc.py](utils/lib_feature_proc.py), in the `class FeatureGenerator`, change the function `def add_cur_skeleton`! 
    
    The function reads in a raw skeleton and outputs the feature generated from this raw skeleton as well as previous skeletons. The feature will then be saved to `features_X.csv` by the script [s3_preprocess_features.py](src/s3_preprocess_features.py) for the next training step.

  * How to include joints of the head?

    You need to change the aforementioned `add_cur_skeleton` function. 
    
    I suggest you to write a new function to extract the head features, and then append them to the returned variable(feature) of `add_cur_skeleton`.

    Please read `def retrain_only_body_joints` in `utils/lib_feature_proc.py` if you want to add the head joints.

  * How to change the classifier to RNN?

    There are two major changes to do:
    
    First, change the aforementioned `add_cur_skeleton`. Instead of manually extracing time-serials features as does by the current script, you may simply stack the input skeleton with previous skeletons and then output it.

    Second, change the `def __init__` and `def predict` function of `class ClassifierOfflineTrain` in [utils/lib_classifier.py](utils/lib_classifier.py) to add an RNN model.


## Main scripts
The 5 main scripts are under `src/`. They are named under the order of excecution:
```
src/s1_get_skeletons_from_training_imgs.py   # deprecated     
src/s12_new.py
src/s2_put_skeleton_txts_to_a_single_txt.py
src/s3_preprocess_features.py
src/s4_train.py 
src/s5_test.py  # deprecated
src/s5_test_new.py
```

The input and output of these files as well as some parameters are defined in the configuration file [config/config.yaml](config/config.yaml). I paste part of it below just to provide an intuition:

``` yaml
classes: ['sit', 'stand', 'walk', 'meal']

image_filename_format: "{:05d}.jpg"
skeleton_filename_format: "{:08d}.json"                                              #MODIFIED
 
features:
  window_size: 5 # Number of adjacent frames for extracting features. 

s1_get_skeletons_from_training_imgs.py:
  openpose:
    model: cmu # cmu or mobilenet_thin. "cmu" is more accurate but slower.
    img_size: 656x368 #  656x368, or 432x368, 336x288. Bigger is more accurate.
  input:
    images_description_txt: data/source_images3/valid_images.txt
    images_folder: data/source_images3/
  output:
    images_info_txt: data_proc/raw_skeletons/images_info.txt
    detected_skeletons_folder: &skels_folder data_proc/raw_skeletons/skeleton_res/
    viz_imgs_folders: data_proc/raw_skeletons/image_viz/

s2_put_skeleton_txts_to_a_single_txt.py:
  input:
    # A folder of skeleton txts. Each txt corresponds to one image.
    detected_skeletons_folder: *skels_folder
  output:
    # One txt containing all valid skeletons.
    all_skeletons_txt: &skels_txt data_proc/raw_skeletons/skeletons_info.txt

s3_preprocess_features.py:
  input: 
    all_skeletons_txt: *skels_txt
  output:
    processed_features: &features_x data_proc/features_X.csv
    processed_features_labels: &features_y data_proc/features_Y.csv

s4_train.py:
  input:
    processed_features: *features_x
    processed_features_labels: *features_y
  output:
    model_path: model/trained_classifier.pickle
```

For how to run the main scripts, please see the Section `4. How to run: Inference` and `6. How to run: Training`.

# 4. How to run: Inference

## Introduction
The script [src/s5_test_new.py](src/s5_test_new.py) is for doing real-time action recognition. 


The classes are set in [config/config.yaml](config/config.yaml) by the key `classes`.

The supported input includes **video file**, **a folder of images**, and **web camera**, which is set by the command line arguments `--data_type` and `--data_path`.

The trained model is set by `--model_path`, e.g.:[model/trained_classifier.pickle](model/trained_classifier.pickle).

The output is set by `--output_folder`, e.g.: output/.

The test data (a video, and a folder of images) are already included under the [data_test/](data_test/) folder.

An example result of the input video "exercise.avi" is:

```
output
├── action.csv
└── action_stabilizzato.csv
```
Also, the video result is saved in the main folder.

Example commands are given below:

## Test on video file
``` bash
python src/s5_test.py \
    --model_path model/trained_classifier.pickle \
    --data_type video \
    --data_path data_test/exercise.avi \
    --output_folder output
```

## Test on a folder of images
``` bash
python src/s5_test.py \
    --model_path model/trained_classifier.pickle \
    --data_type folder \
    --data_path data_test/apple/ \
    --output_folder output
```

## Test on web camera
``` bash
python src/s5_test.py \
    --model_path model/trained_classifier.pickle \
    --data_type webcam \
    --data_path 0 \
    --output_folder output
```

# 5. Training data


## Download my data
Follow the instructions in [data/download_link.md](data/download_link.md) to download the data. Or, you can create your own. The data and labelling format are described below.

## Data format

Each data subfolder (e.g. `data/source_images3/jump_03-02-12-34-01-795/`) contains images named as `00001.jpg`, `00002.jpg`, etc.   
The naming format of each image is defined in [config/config.yaml](config/config.yaml) by the sentence: `image_filename_format: "{:05d}.jpg"`.

The images to be used as training data and their label are configured by this txt file: [data/source_images3/valid_images.txt](data/source_images3/valid_images.txt).  
A snapshot of this txt file is shown below:
```
jump_03-02-12-34-01-795
52 59
72 79

kick_03-02-12-36-05-185
54 62
```
In each paragraph,  
the 1st line is the data folder name, which should start with `"${class_name}_"`. 
The 2nd and following lines specify the `staring index` and `ending index` of the video that corresponds to that class.

Let's take the 1st paragraph of the above snapshot as an example: `jump` is the class, and the frames `52~59` & `72~79` of the video are used for training.

## Classes

The classes are set in [config/config.yaml](config/config.yaml) under the key word `classes`. No matter how many classes you put in the training data (set by the folder name), only the ones that match with the classes in **config.yaml** are used for training and inference.



# 6. How to run: Training

First, you may read
* Section `5. Training data`
* Section `3. Program structure`
* [config/config.yaml](config/config.yaml)

to know the training data format and the input and output of each script.

Then, follow the following steps to do the training:
* Collect your own data and label them, or use my data. [Here](https://github.com/felixchenfy/record_images_from_usbcam) is tool to record images from web camera.
* If you are using your data, change the values of `classes` and `images_description_txt` and `images_folder` inside [config/config.yaml](config/config.yaml).
* Depend on your need, you may change parameters in [config/config.yaml](config/config.yaml).
* Finally, run the following scripts one by one:
    ``` bash
    python src/s1_get_skeletons_from_training_imgs.py
    python src/s2_put_skeleton_txts_to_a_single_txt.py 
    python src/s3_preprocess_features.py
    python src/s4_train.py 
    ```

By default, the intermediate data are saved to [data_proc/](data_prco/), and the model is saved to [model/trained_classifier.pickle](model/trained_classifier.pickle).  
After training is done, you can run the inference script `src/s5_test.py` as described in Section `4. How to run: Inference`.

# 7. Result and Performance

Unfortunately this project only works well on myself, because I only used the video of myself.

The performance is bad for (1) people who have different body shape, (2) people are far from the camera. **How to improve?** I guess the first thing to do is to collect larger training set from different people. Then, improve the data augmentation and featuer selection.

Besides, my simple tracking algorithm only works for a few number of people (maybe 5). 

Due to the not-so-good performance of action recognition, I guess you can only use this project for course demo, but not for any commercial applications ... T.T 
