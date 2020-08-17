# Bootstrap Your Own Latent

This is the implementation of the pre-training and linear evaluation pipeline of
BYOL - https://arxiv.org/abs/2006.07733.

Using this implementation should achieve a top-1 accuracy on Imagenet between
74.0% and 74.5% after about 8h of training using 512 Cloud TPU v3.

The main pretraining module is `byol_experiment.py`. By default it uses BYOL to
pretrain a Resnet-50 on Imagenet. In parallel, we train a classifier on top of
the representation to assess its performance during training. This classifier
does not back-propagate any gradient to the ResNet-50.

The evaluation module is `eval_experiment.py`. It evaluates the performance of
the representation learnt by BYOL (using a given checkpoint).

## Setup

To set up a Python virtual environment with the required dependencies, run:

```shell
python3 -m venv byol_env
source byol_env/bin/activate
pip install --upgrade pip
pip install -r byol/requirements.txt
```

The code uses `tensorflow_datasets` to load the Imagenet dataset. Manual
download may be required; see https://www.tensorflow.org/datasets/catalog/imagenet2012
for details.

## Running a short pre-training locally

To run a short (40 epochs) pre-training experiment on a local machine, use:

```shell
mkdir /tmp/byol_checkpoints
python -m byol.main_loop \
  --experiment_mode='pretrain' \
  --worker_mode='train' \
  --checkpoint_root='/tmp/byol_checkpoints' \
  --pretrain_epochs=40
```

## Full pipeline and presets

The various parts of the pipeline can be run using:

```shell
python -m byol.main_loop \
  --experiment_mode=<'pretrain' or 'linear-eval'> \
  --worker_mode=<'train' or 'eval'> \
  --checkpoint_root=</path/to/the/checkpointing/folder> \
  --pretrain_epochs=<40, 100, 300 or 1000>
```

### Pretraining
Setting `--experiment_mode=pretrain` will configure the main loop for
pretraining; we provide presets for 40, 100, 300 and 1000 epochs.

Use `--worker_mode=train` for a training job, which will regularly save
checkpoints under `<checkpoint_root>/pretrain.pkl`. To monitor the progress of
the pretraining, you can run a second worker (using `--worker_mode=eval`) with
the same `checkpoint_root` setting. This worker will regularly load the
checkpoint and evaluate the performance of a linear classifier (trained by the
pretraining `train` worker) on the `TEST` set.

Note that the default settings are set for large-scale training on Cloud TPUs,
with a total batch size of 4096. To avoid the need to re-run the full
experiment, we will make a pre-trained checkpoint available on GCP.

### Linear evaluation
Setting `--experiment_mode=linear-eval` will configure the main loop for
linear evaluation; we provide presets for 80 epochs.

Use `--worker_mode=train` for a training job, which will load the encoder
weights from an existing checkpoint (form a pretrain experiment) located at
`<checkpoint_root>/pretrain.pkl`, and train a linear classifier on top of this
encoder. The weights from the linear classifier trained in the pretraining phase
will be discarded.

The training job will regularly save checkpoints under
`<checkpoint_root>/linear-eval.pkl`. You can run a second worker
(using `--worker_mode=eval`) with the same `checkpoint_root` setting to
regularly load the checkpoint and evaluate the performance of the classifier
(trained by the linear-eval `train` worker) on the test set.

Note that the above will run a simplified version of the linear evaluation
pipeline described in the paper, with a single value of the base learning rate
and without using a validation set. To fully reproduce the results from the
paper, one should run 5 instances of the linear-eval `train` worker (using only
the `TRAIN` subset, and each using a different `checkpoint_root`), run the
`eval` worker (using only the `VALID` subset) for each checkpoint, then run a
final `eval` worker on the `TEST` set.


## Running on GCP

Notice: we currently do not recommend running the full experiment on public
Cloud TPUs. We provide an alternative small-scale GPU setup in the next section.

The experiments from the paper were run on Google's internal infrastructure.
Unfortunately, training JAX models on publicly available Cloud TPUs is currently
in its early stages; in particular, we found the learning to be heavily
bottlenecked by the data loading pipeline, resulting in a significantly slower
training. We will update the code and instructions below once the limitation is
addressed.

To train the model on TPU using Google Compute Engine, please follow the
instructions at https://cloud.google.com/tpu/docs/imagenet-setup to first
download and preprocess the ImageNet dataset and upload it to a
Cloud Storage Bucket. From a GCE VM, you can check that `tensorflow_datasets`
can correctly load the dataset by running:

```python
import tensorflow_datasets as tfds
tfds.load('imagenet2012', data_dir='gs://<your-bucket-name>')
```

To learn how to use JAX with Cloud TPUs, please follow the instructions here:
https://github.com/google/jax/tree/master/cloud_tpu_colabs.


## Setup for fast iteration

In order to make reproduction easier, it is possible to change the training
setup to use the smaller [imagenette](https://github.com/fastai/imagenette)
dataset (9469 training images with 10 classes). The following setup and
hyperparameters can be used on a machine with a single V100 GPU:

- in `utils/dataset.py`:
  - update `Split.num_examples` with the figures from [tfds](https://www.tensorflow.org/datasets/catalog/imagenette) (with `Split.VALID: 0`)
  - use `imagenette/160px-v2` in the call to `tfds.load`
  - use 128x128 px images (_i.e._, replace all instances of `224` by `128`)
  - it doesn't seem necessary to change the color normalization (make sure to
    not replace the value `0.224` by mistake in the previous step).
- in `configs/byol.py`, use:
  - `num_classes`: `10`
  - `network_config.encoder`: `ResNet18`
  - `optimizer_config.weight_decay`: `1e-6`
  - `lr_schedule_config.base_learning_rate`: `2.0`
  - `evaluation_config.batch_size`: `25`
  - other parameters unchanged.

You can then run:

```shell
mkdir /tmp/byol_checkpoints
python -m byol.main_loop \
  --experiment_mode='pretrain' \
  --worker_mode='train' \
  --checkpoint_root='/tmp/byol_checkpoints' \
  --batch_size=256 \
  --pretrain_epochs=1000
```

With these settings, BYOL should achieve ~92.3% top-1 accuracy (for the
*online* classifier) in roughly 4 hours. Note that the above parameters were not
finely tuned and may not be optimal.