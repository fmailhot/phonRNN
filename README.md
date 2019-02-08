# PhonRNN

## Purpose

This repository contains code for running an LSTM network for the purposes of modeling word-level phonotactics, and analyzing the output of this modeling.

## Dependencies

* LSTM modeling: Python 3.4+, with the following packages
  * [pytorch](https://pytorch.org/)
  * [pronouncing](https://github.com/aparrish/pronouncingpy)
  * [numpy](http://www.numpy.org/)
  * [argparse](https://github.com/ThomasWaldmann/argparse/)
  * CUDA is also highly recommended for speed, as is a decent GPU
* Running ngram baselines:
  * [SRiLM](http://www.speech.sri.com/projects/srilm/download.html)
  * Python 2.7+, with the following packages:
    - [swig-srilm](https://github.com/desilinguist/swig-srilm)
    - [argparse](https://github.com/ThomasWaldmann/argparse/)
* Analysis and visualization: R, with the following packages
  - cluster
  - plyr
  - reshape2
  - tidyr
  - ggplot2
  - ggdendro
  - gplots
  - RColorBrewer
  - graphics

## Organization

* `data` contains the source data to run/evaluate the model on
  - `src` contains the preprocessing scripts that were used to get the data into a workable form for the model:
    - `CELEX2.py` is for processing `.cd` files from [the CELEX2 database](https://catalog.ldc.upenn.edu/LDC96L14). In our experiment, we used the `epl.cd` from CELEX2 as input to this script.
    - `Daland-et-al_2011.py` is for processing nonwords from Daland et al., 2011. Many thanks to Robert Daland for making this data available [on his website](https://sites.google.com/site/rdaland/publications/Daland_etal_2011__AverageScores.csv?attredirects=0&d=1).
  - `processed` contains the result of the scripts from `scripts`. `CELEX2/lemmas` is what we used for training and testing the models in the experiment.
* `baselines` contains ratings of words from other models that are not the neural network model
  * `Daland-et-al_2011` contains results from human judgments of non-words, collected by Daland et al., 2011 (many thanks to Robert Daland for making this data available [on his website](https://sites.google.com/site/rdaland/publications/Daland_etal_2011__AverageScores.csv?attredirects=0&d=1))
  * `ngram`
    * `src`: You will need Python 2.7 to run these scripts, because they rely on the [swig-SRILM script](https://github.com/desilinguist/swig-srilm). Install `srilm.py` into this directory, and use this as your working directory to run `ngram-baseline.py`, which calculates ngram probabilities for all words in a corpus
    * `results/CELEX2` contain results of running the ngram model on our source data, which was located at `/data/processed/CELEX2/lemmas` 
      * `9gram.txt`  was generated by running SRiLM's `ngram-count` using `/data/processed/CELEX2/lemmas/train.txt` as the `-text`, with no smoothing and `-order 9` 
      * `9gram-wb.txt` was generated in the same manner, but using the `-wb` flag to trigger Witten-Bell smoothing.
* `src` contains the files needed to instantiate our model, as well as run a grid-search over hyperparameters. Usage is explained in the "How-To" section of this document.
* `results` was the data used for analysis in the experiment.
  * If you decide to run the model on your own, this is what will be generated:
    * `final_eval-agg.csv` contains log probabilities of each word in the training, test, and validation corpora from the experiment
    * `summary.csv` contains data generated during the training phase of each run of the model. Each row in this file represents one epoch of training.
    * You will notice several subdirectories, whose names are of the form `[number1]-[number2]` . Each subdirectory here is a random initialization, or a "run" of the model. `number1` indexes the condition (whose precise description is given in `summary.csv`), and `number2` indexes the initialization. Inside each directory is the following:
      * `best-model.pt` is the `.pt` file of the model at the point in training when it assignes the highest probability to the validation set
      * `emb-before.txt` is the phone embedding before training (for feature-aware conditions, these will be feature matrices)
      * `emb-after.txt` is the phone embedding after training
      * `sample.txt` is a sample of text generated from the model
      * `random-reset.data` saves the model activations before it's been exposed to any words. In the results I've provided, these weights are zeroed out, but you may wish to play around with these.
  * `daland-probs.csv` contains log probabilities of each word in the Daland et al., 2011 corpus, as computed by each initialization of the models. It was generated by running `/src/eval.py` on `/data/processed/Daland-et-al_2011/test.txt`
* `analysis` contains R files that interact with the results of experiments. You can use these to generate your own pretty plots, or to check my work. The easiest file to start with here will be `analyze.Rmd`.

## How-To

Before anything, make sure your data is in the correct format for the experiment scripts to read it! All data files should have one word per line, with spaces separating each symbol (phonetic segment) in the word. Example:

```
DH I S
I Z
AH
S M AA L
EH K S AE M P L
D EY T AH S EH T
```

Symbols can either be in IPA or ARPABET format, but you'll have to specify which one when you run the model.

A corpus should have these three data files:

1. `train.txt` = the file to train the model on
2. `valid.txt` = the file to use as validation set
3. `test.txt` = the file to use as a test set

For an example of a well-formed corpus, see `/data/processed/CELEX2/lemmas`

### Running the model once

Run `src/main.py` with the following parameters:

| flag                | type    | description                                                  |
| ------------------- | ------- | ------------------------------------------------------------ |
| `--data`            | string  | location of the data corpus                                  |
| `--model`           | string  | type of recurrent net (possible values are "RNN_TANH", "RNN_RELU", "LSTM", "GRU") |
| `--phonol_emb`      | boolean | use phonological embedding as a starting point               |
| `--fixed_emb`       | boolean | don't change embedding weights                               |
| `--emsize`          | integer | size (length) of phone embedding                             |
| `--nhid`            | integer | number of hidden units per layer                             |
| `--nlayers`         | integer | number of recurrent layers                                   |
| `--lr`              | integer | initial learning rate                                        |
| `--anneal_factor`   | float   | amount by which to anneal learning rate if no improvement on annealing criterion set (1 = no annealing, 0.5 = learning rate halved) |
| `--anneal_train`    | boolean | anneal learning rate using the training loss instead of the validation loss |
| `--patience`        | integer | number of training epochs to wait for validation loss to improve before annealing learning rate |
| `--clip`            | float   | amount of gradient clipping to employ (a maximum cap on gradients) |
| `--epochs`          | integer | upper epoch limit                                            |
| `--dropout`         | float   | dropout applied to layers (0 = no dropout)                   |
| `--tied`            | boolean | tie phone embedding and softmax weights                      |
| `--seed`            | integer | random seed                                                  |
| `--cuda`            | boolean | use cuda                                                     |
| `--stress`          | boolean | keep track of word stress                                    |
| `--log-interval`    | integer | interval at which to print a log                             |
| `--save_dir`        | string  | directory in which to save various characteristics of the final model |
| `--summary`         | string  | where to save the summary CSV, within `--save_dir`           |
| `--condition`       | integer | Condition index, referenced in summary CSV                   |
| `--run`             | integer | Run index, within condition                                  |
| `--feat_tree`       | string  | Feature tree to use. At present, only 'Futrell' or 'orig' are possible, and 'Futrell' is highly recommended |
| `--alphabet`        | string  | Format that the data is in (IPA or ARPABET)                  |
| `--set_unset_nodes` | boolean | Use set/unset nodes                                          |
| `--random_reset`    | boolean | Reset the model's activations to an initial random state after each word. |

So a command might look something like:

```
$ python src/main.py --data data/processed/CELEX2/lemmas --model LSTM --phonol_emb --cuda
```

### Running an experiment

To run an experiment (i.e., train several models while running a single command, and keep track of them), edit lines 31-51 of of `src/grid_search.py` with the model parameters you'd like to test. At present, these are set up as they were for our experiment: ready for replication!

You can also change some of the parameters of the experiment within the call itself:

| flag                 | type    | description                                                  |
| -------------------- | ------- | ------------------------------------------------------------ |
| `--data`             | string  | location of the data corpus                                  |
| `--alphabet`         | string  | Format that the data is in (IPA or ARPABET)                  |
| `--feat_tree`        | string  | Feature tree to use. At present, only 'Futrell' or 'orig' are possible, and 'Futrell' is highly recommended |
| `--condition_runs`   | integer | Runs per condition                                           |
| `--output_dir`       | string  | path to save results, including summary CSV and model checkpoints |
| `--summary_filename` | string  | path to save summary CSV, within results directory           |
| `--cuda`             | boolean | use CUDA                                                     |
| `--run_start`        | integer | Where to start seeding the model                             |

### Evaluating the model

The `src/eval.py` script is used for evaluation. You can evaluate your models one at a time, or a whole group of models (as from an experiment, described above) at once. Similarly, you can test just a single file at once, or a whole corpus directory. Here are all the flags that can be used with this script.

| flag             | type    | description                                                  |
| ---------------- | ------- | ------------------------------------------------------------ |
| `--data`         | string  | location of the data corpus                                  |
| `--data_file`    | string  | location of a single test file **(will override `--data`)**  |
| `--checkpoint`   | string  | single model checkpoint to test                              |
| `--batch_dir`    | string  | results directory to use for batch testing **(will override `--checkpoint`)** |
| `--summary_file` | string  | file name of the summary file, within the batch_dir          |
| `--out`          | string  | name of output file to write the results to                  |
| `--cuda`         | boolean | use CUDA                                                     |
| `--seed`         | integer | random seed                                                  |
| `--stress        | boolean | keep track of word stress                                    |

