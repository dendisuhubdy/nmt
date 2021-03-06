# Neural machine translation

Repository to collect code for neural machine translation internally at MILA.
The short-term objective is to have an attention-based model working on
multiple GPUs (see [#6](https://github.com/bartvm/nmt/issues/6)). My proposal
is to base the model code of Cho's for now (see
[#1](https://github.com/bartvm/nmt/issues/1), because it has simpler internals
than Blocks that we can hack away at if needed for multi-GPU.

To have a central collection of research ideas and discussions, please create
issues and comment on them.

## Setting up your environment

To run these experiments you need at minimum an environment as described
in `environment.yml`.

## Training on the lab computers

To train efficiently, make sure of the following:

* Use cuDNN 4; if cuDNN is disabled it will take the gradient of the
  softmax on the CPU which is much slower.
* Enable CNMeM (e.g. add `cnmem = 0.98` in the `[lib]` section of your
  `.theanorc`).

Launching with Platoon can be done using `platoon-launcher nmt gpu0 gpu1
-c="config.json 4"` where 4 is the number of workers.. To watch the logs
it's wortwhile to alias the command `watch tail "$(ls -1dt
PLATOON_LOGS/nmt/*/ | head -n 1)*"`.

Starting a single GPU experiment is done with `python nmt_single.py
config.json`.

## Training on Helios

To submit jobs on Helios, submit the `nmt.pbs` file using e.g.

```bash
msub nmt.pbs -F "\"config.json\"" -l nodes=1:gpus=2 -l walltime=1:00:00
```

Note that by default K20 GPUs are assigned for multi-GPU experiments.
K80s usually have a higher availability. They can be requested by adding
`-l feature=k80`.

This submission script does the following:

* Read data from a shared directory, `$RAP/nmt`
* Set `THEANO_FLAGS`
* Import Theano to make sure that everything works
* Pick random ports for communication, batches, and the log. This way
  multiple jobs on the same node don't interfere with each other.

It assumes that your Python installation is contained in
`$HOME/miniconda3`. If it is elsewhere, either change `nmt.pbs` or
change your `PATH` in your `.bashrc`.

## WMT16 data

A quick overview on downloading and preparing the [WMT16
data](http://www.statmt.org/wmt16/translation-task.html), using
English-German as an example.

```bash
# Check on the website which datasets are available for the language pair
cat <<EOF | xargs -n 1 -P 4 wget -q
http://www.statmt.org/wmt13/training-parallel-europarl-v7.tgz
http://www.statmt.org/wmt13/training-parallel-commoncrawl.tgz
http://data.statmt.org/wmt16/translation-task/training-parallel-nc-v11.tgz
EOF

# Unpack
ls *.tgz | xargs -I {} tar xvfz {}

# Merge all the files into one
cat commoncrawl.de-en.de training/europarl-v7.de-en.de \
    training-parallel-nc-v11/news-commentary-v11.de-en.de > wmt16.de-en.de
cat commoncrawl.de-en.en training/europarl-v7.de-en.en \
    training-parallel-nc-v11/news-commentary-v11.de-en.en > wmt16.de-en.en
```

We perform minimal preprocessing similar to the [Moses baseline
system](http://www.statmt.org/moses/?n=Moses.Baseline). We then shuffle the
data so that all the corpora are mixed.

```bash
MOSES=/path/to/mosesdecoder
LANG1=de
LANG2=en

source data.sh

# For e.g. TED data, call strip

tokenize wmt16.de-en.en
tokenize wmt16.de-en.de

truecase wmt16.de-en.tok.en
truecase wmt16.de-en.tok.de

# For monolingual data, skip the cleaning step
clean wmt16.de-en.tok.true

# For monolingual data, just use `shuf infile > outfile`
shuffle wmt16.de-en.tok.true.clean
```

Count the words and create a vocabulary.

```bash
create-vocabulary wmt16.de-en.tok.true.clean.shuf.en > wmt16.de-en.vocab.en
create-vocabulary wmt16.de-en.tok.true.clean.shuf.de > wmt16.de-en.vocab.de
```

Truecase the validation sets use the model trained on the parallel data.

```bash
truecase newstest2013.de wmt16.de-en.truecase-model.de
truecase newstest2013.en wmt16.de-en.truecase-model.en
```
