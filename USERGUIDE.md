## User Guide

#### Video Browser with Keyword Search

A web interface is provided for output created by the `vids2web` command in the VM. You can view it from the host computer in a Chrome or Safari web browser at the URL http://192.168.56.101/ (or if running on AWS, the URL shown after you first provisioned the machine with `vagrant up`)

Content will be updated automatically when you run `vids2web.sh` from the path `~/tools/eesen-offline-transcriber` in the VM, or manually if you put new .mp4 videos in the `/vagrant/www/video/` folder, and .srt subtitles (with matching base name) in the `/vagrant/www/sub/` folder, and run `mkpages.sh`.

`mkpages.sh` (re-)generates the main page (index.html) and player pages for each video (playXX.html, where XX is a video or audio file base name) This will create thumbnails and .vtt format subtitles for the web video player 

#### Customizing the VM

You can also log directly into the VM with `vagrant ssh` and look around. For example, change directories to `/home/vagrant/tools/eesen-offline-transcriber` to find README instructions there. You can initiate transcription from here with `speech2text.sh` and output will appear in `build/output`. You can run scripts that queue several files for transcription with `batch.sh`

Note that the size of the VM is controlled by the Vagrantfile, and asks for 2 CPUs and 8GB RAM:

    vbox.cpus = 2
    vbox.memory = 8192

This supports transcribing of audio/video files with small utterance lengths*. But for long utterances you may need to crank this to 8-12 GB, which means your host computer may need as much as 16 GB. Either you will need more RAM on the host you run locally using VirtualBox, or you will need to specify a more powerful `aws.instance.type` in Vagrantfile. [AWS Instance Types](https://aws.amazon.com/ec2/instance-types/) Similarly, if you want to transcribe several things in parallel, you'll want to crank up the number pf `Procs` at the end of `/etc/slurm-llnl/slurm.conf` as well as increasing the RAM allocated to the VM. An example, setting CPUs to 8, RAM to 30000, and Procs to 7 seems to work pretty well on a 32 GB 8 core host. 

cd /home/vagrant/tools/eesen-offline-transcriber
Initiate transcription of the test file test2.mp3 with ./speech2text.sh /vagrant/test2.mp3
Output should appear in build/output/test2.*

#### 8khz Models

We cannot give out the models trained on non-open-source data, but if you are able to obtain them, you can run the system in 8k mode, by changing the commented out lines in `Makefile.options`
```
# 8k models from switchboard
ACWT=0.6
GRAPH_DIR?=$(EESEN_ROOT)/asr_egs/swbd/v1-pitch/data/lang_phn_sw1_fsh_tgpr
#GRAPH_DIR?=$(EESEN_ROOT)/asr_egs/swbd/v1-pitch/data/lang_phn_sw1_tg
MODEL_DIR?=$(EESEN_ROOT)/asr_egs/swbd/v1-pitch/exp/train_phn_l5_c320
sample_rate=8k
fbank=make_fbank_pitch
```

#### Watched Folder Automatic Transcription
The VM watches the shared host folder `transcribe_me`. Any files placed in this folder get queued as transcription jobs, and results appear in the same folder with extension `.ctm`. Results also automatically populate the video browser web page. Log files appear in `log/` by job number. If you want to disable this behavior, comment out the line in `Vagrantfile` that runs `watch.sh` before running `vagrant up`, or kill the watch.sh process in the VM.

#### *Segmentation and Utterance Lengths

Have a look in `Makefile` at the definition of `SEGMENTS`. 
```
# Some audio produces no results (can't be segmented), or transcribes better
# when segmented differently
# Changing SEGMENTS to one of these values gives more flexibilty;
# (see http://www-lium.univ-lemans.fr/diarization/doku.php/quick_start)
#
#    show.seg       : default - final segmentation with NCLR/CE clustering
#    show.i.seg     : initial segmentation (entire audio)
#    show.pms.seg   : sPeech/Music/Silence segmentation (don't use)
#    show.s.seg     : GLR based segmentation, make Small segments
#    show.l.seg     : linear clustering (merge only side by side segments)
#    show.h.seg     : hierarchical clustering
#    show.d.seg     : viterbi decoding
#    show.adj.h.seg : boundaries adjusted
#    show.flt1.seg  : filter spk segmentation according to pms segmentation
#    show.flt2.seg  : filter spk segmentation according to pms segmentation
#    show.spl.seg   : segments longer than 20 sec are split
#    show.spl10.seg : segments longer than 10 sec are split
#    show.g.seg     : the gender and the bandwith are detected
SEGMENTS ?= show.seg
```

The default segmentation strategy done by LIUM is `show.seg` but we override it in `Makefile.options` with `show.s.seg` to produce the maximum number of small segments, with the side effect of also assuming there is only one speaker.  (no per-speaker MFCC calculations) 

If you can provide your own segmentation, this may
improve upon the default LIUM segmenter that is part of EESEN transcriber.

1. In the VM, path to `/home/vagrant/tools/eesen-offline-transcriber`

2. Place a video or audio file and segmentation file in folder `src-audio/`,
with the same base name, and extensions `.mp4` and `.seg`, e.g. for base name `myvideo`

  ```
  src-audio/myvideo.mp4 
  src-audio/myvideo.seg
  ```
The `.seg` segmentation file you provide must have the format:
  ```
  column 1 is the base name of the video file,
  column 2 is always “1”
  column 3 is the start time of the segment in hundredths of a second
  column 4 is the length of the segment in hundredths of a second
  the remaining columns are always “U U U S0″
```
  Example `myvideo.seg`:
  ```
  HVC037 1 4 2770 U U U S0 
  HVC037 1 2774 490 U U U S0 
  HVC037 1 3264 12930 U U U S0
  ```
3. Run the script `run-segmented.sh` with the basename as an argument, e.g.:
  ```
  ./run-segmented.sh HVC037
  ```
This produces by default subtitle `.srt` files. If you want another format, edit the
last line of `run-segmented.sh` to specify the desired extension.

##### Scoring

Standard NIST sclite scoring is supported for data in .sph and .stm format via the `run-scored.sh` and `run-scored-8k` scripts.

##### TEDLIUM data

To run the full Eesen TEDLIUM experiment, you will need the TEDLIUM data, which must be
downloaded separately.  It is beyond the scope of this document to describe the full
Eesen TEDLIUM experiment, since that is part of a different repository [here](http://github.com/srvk/eesen).
However the basic idea is that there is a working folder in the Eesen system for running
the TEDLIUM experiment: `~/eesen/asr_egs/tedlium/v2-30ms`. Beneath this folder is a `db`
folder into which 23 GB of TEDLIUM corpus data gets downloaded. The experiment scripts will
attempt to download the data if it does not exist, but if you already have it on your
host computer, it's convenient to make the `db` folder be a symbolic link to `/vagrant/db`
into which you can place a copy of the data - this is a shared folder on the host computer
where you launched the VM, typically ending in `eesen-transcriber`

##### Models

Also in `Makefile.options` are paths (in the VM) to the models used for decoding. If you create a new acoustic model (see Language Remodeling below), you will want to change this to point to your new model. A recent update provides models designed for 30ms frame sizes, resulting in much faster decoding.  (1/3-1/7x real time, depending on configuration)

#### Improving Word Error Rate: rescoring with large LM

It's possible to get a 2% relative improvement in WER by adding a step to the decoding process, which rescores intermediate lattices with a larger language model (Cantab 4-gram, unpruned). This adds extra time to the decoding process, and requires more memory, so it is commented out of the Vagrantfile by default. But if you have a 16 GB host machine, and the extra time, try it out. The data file `rescore-eesen.tgz` contains an extra program, the language model, and a patched Makefile.

    ```
    cd /home/${user}
    wget http://speechkitchen.org/vms/Data/rescore-eesen.tgz
    tar zxvf rescore-eesen.tgz
    rm rescore-eesen.tgz    
    ```
### Word Level Alignments
To get word-level alignments for transcriptions produced by the transcriber, first transcribe something. Let's use the example, `test2.mp3`. Then run the script run_align.sh. This script makes assumptions about the decoding graph to use. To use the settings from `vagrant/Makefile.options` instead, run `make build/output/<basename>.ali`

```
./speech2text /vagrant/test2.mp3
./run_align.sh test2
```
Results will be placed alongside outputs in `build/output`, in this case `build/output/test2.ali` so for example:
```
cat build/output/test2.ali
test2-S0---0000.090-0013.900 1 0.00 0.24 things 
test2-S0---0000.090-0013.900 1 0.24 0.07 will 
test2-S0---0000.090-0013.900 1 0.31 0.16 change 
test2-S0---0000.090-0013.900 1 0.47 0.05 in 
test2-S0---0000.090-0013.900 1 0.52 0.28 ways 
test2-S0---0000.090-0013.900 1 0.80 0.05 that 
test2-S0---0000.090-0013.900 1 0.85 0.09 they're 
test2-S0---0000.090-0013.900 1 0.94 0.20 fragile 
test2-S0---0000.090-0013.900 1 1.14 0.33 environment 
test2-S0---0000.090-0013.900 1 1.47 0.14 simply 
test2-S0---0000.090-0013.900 1 1.61 0.11 can't 
test2-S0---0000.090-0013.900 1 1.72 0.40 support 
test2-S0---0000.090-0013.900 1 2.12 0.04 and 
test2-S0---0000.090-0013.900 1 2.16 0.07 that 
test2-S0---0000.090-0013.900 1 2.23 0.08 leads 
test2-S0---0000.090-0013.900 1 2.31 0.03 to 
test2-S0---0000.090-0013.900 1 2.34 0.25 starvation 
test2-S0---0000.090-0013.900 1 2.59 0.05 it 
test2-S0---0000.090-0013.900 1 2.64 0.09 leads 
test2-S0---0000.090-0013.900 1 2.73 0.03 to 
test2-S0---0000.090-0013.900 1 2.76 0.20 uncertainty 
test2-S0---0000.090-0013.900 1 2.96 0.04 at 
test2-S0---0000.090-0013.900 1 3.00 0.11 leads 
test2-S0---0000.090-0013.900 1 3.11 0.46 unrest 
test2-S0---0000.090-0013.900 1 3.57 0.14 so 
test2-S0---0000.090-0013.900 1 3.71 0.06 the 
test2-S0---0000.090-0013.900 1 3.77 0.16 climate 
test2-S0---0000.090-0013.900 1 3.93 0.11 change 
test2-S0---0000.090-0013.900 1 4.04 0.10 is 
test2-S0---0000.090-0013.900 1 4.14 0.06 will 
test2-S0---0000.090-0013.900 1 4.20 0.05 be 
test2-S0---0000.090-0013.900 1 4.25 0.14 terrible 
test2-S0---0000.090-0013.900 1 4.39 0.05 for 
test2-S0---0000.090-0013.900 1 4.44 0.16 them 
```

### Cleaning Up

Sometimes it helps to know how to shut down, as well as install and run a system. Two use cases come to mind.

Shutting down the virtual machine:

    vagrant halt

Cleaning files associated with the embedded VirtualBox virtual machine (i.e. wipe everything)

    vagrant destroy
    
This will not wipe out any local data or results, only the virtual machine (either VirtualBox locally or the AWS AMI)

### Language Remodeling

This VM now supports language model building according to instructions at SpeechKitchen.org: [Kaldi Language Model Building](http://speechkitchen.org/kaldi-language-model-building/)

### Error Analysis

If you ran the full EESEN TEDLIUM experiment (`~/eesen/asr_egs/tedlium/v1`), it is possible to use SpeechKitchen.org's [Error Analysis Page](http://speechkitchen.org/error-analysis-instructions-for-tedlium-vm/) to produce and view graphs and charts that let you play with the scoring data and visualize it in different ways. (Look near the end for EESEN specifics)

### Tips & Tricks

You might want to install another useful vagrant plugin such as [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) which automatically synchronizes the VirtualBox Guest Additions in the VM

    vagrant plugin install vagrant-vbguest
