.. 
    Copyright 2020 The HuggingFace Team. All rights reserved.

    Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
    the License. You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
    an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
    specific language governing permissions and limitations under the License.

Fine-tuning a pretrained model
=======================================================================================================================

In this tutorial, we will show you how to fine-tune a pretrained model from the Transformers library. In TensorFlow,
models can be directly trained using Keras and the :obj:`fit` method. In PyTorch, there is no generic training loop so
the 🤗 Transformers library provides an API with the class :class:`~transformers.Trainer` to let you fine-tune or train
a model from scratch easily. Then we will show you how to alternatively write the whole training loop in PyTorch.

Before we can fine-tune a model, we need a dataset. In this tutorial, we will show you how to fine-tune BERT on the
`IMDB dataset <https://www.imdb.com/interfaces/>`__: the task is to classify whether movie reviews are positive or
negative. For examples of other tasks, refer to the :ref:`additional-resources` section!

.. _data-processing:

Preparing the datasets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <iframe width="560" height="315" src="https://www.youtube.com/embed/_BZearw7f0w" title="YouTube video player"
   frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope;
   picture-in-picture" allowfullscreen></iframe>

We will use the `🤗 Datasets <https://github.com/huggingface/datasets/>`__ library to download and preprocess the IMDB
datasets. We will go over this part pretty quickly. Since the focus of this tutorial is on training, you should refer
to the 🤗 Datasets `documentation <https://huggingface.co/docs/datasets/>`__ or the :doc:`preprocessing` tutorial for
more information.

First, we can use the :obj:`load_dataset` function to download and cache the dataset:

.. code-block:: python

    from datasets import load_dataset

    raw_datasets = load_dataset("imdb")

This works like the :obj:`from_pretrained` method we saw for the models and tokenizers (except the cache directory is
`~/.cache/huggingface/dataset` by default).

The :obj:`raw_datasets` object is a dictionary with three keys: :obj:`"train"`, :obj:`"test"` and :obj:`"unsupervised"`
(which correspond to the three splits of that dataset). We will use the :obj:`"train"` split for training and the
:obj:`"test"` split for validation.

To preprocess our data, we will need a tokenizer:

.. code-block:: python

    from transformers import AutoTokenizer

    tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")

As we saw in :doc:`preprocessing`, we can prepare the text inputs for the model with the following command (this is an
example, not a command you can execute):

.. code-block:: python

    inputs = tokenizer(sentences, padding="max_length", truncation=True)

This will make all the samples have the maximum length the model can accept (here 512), either by padding or truncating
them.

However, we can instead apply these preprocessing steps to all the splits of our dataset at once by using the
:obj:`map` method:

.. code-block:: python

    def tokenize_function(examples):
        return tokenizer(examples["text"], padding="max_length", truncation=True)

    tokenized_datasets = raw_datasets.map(tokenize_function, batched=True)

You can learn more about the map method or the other ways to preprocess the data in the 🤗 Datasets `documentation
<https://huggingface.co/docs/datasets/>`__.

Next we will generate a small subset of the training and validation set, to enable faster training:

.. code-block:: python

    small_train_dataset = tokenized_datasets["train"].shuffle(seed=42).select(range(1000)) 
    small_eval_dataset = tokenized_datasets["test"].shuffle(seed=42).select(range(1000)) 
    full_train_dataset = tokenized_datasets["train"]
    full_eval_dataset = tokenized_datasets["test"]

In all the examples below, we will always use :obj:`small_train_dataset` and :obj:`small_eval_dataset`. Just replace
them by their `full` equivalent to train or evaluate on the full dataset.

.. _trainer:

Fine-tuning in PyTorch with the Trainer API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <iframe width="560" height="315" src="https://www.youtube.com/embed/nvBXf7s7vTI" title="YouTube video player"
   frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope;
   picture-in-picture" allowfullscreen></iframe>

Since PyTorch does not provide a training loop, the 🤗 Transformers library provides a :class:`~transformers.Trainer`
API that is optimized for 🤗 Transformers models, with a wide range of training options and with built-in features like
logging, gradient accumulation, and mixed precision.

First, let's define our model:

.. code-block:: python

    from transformers import AutoModelForSequenceClassification

    model = AutoModelForSequenceClassification.from_pretrained("bert-base-cased", num_labels=2)

This will issue a warning about some of the pretrained weights not being used and some weights being randomly
initialized. That's because we are throwing away the pretraining head of the BERT model to replace it with a
classification head which is randomly initialized. We will fine-tune this model on our task, transferring the knowledge
of the pretrained model to it (which is why doing this is called transfer learning).

Then, to define our :class:`~transformers.Trainer`, we will need to instantiate a
:class:`~transformers.TrainingArguments`. This class contains all the hyperparameters we can tune for the
:class:`~transformers.Trainer` or the flags to activate the different training options it supports. Let's begin by
using all the defaults, the only thing we then have to provide is a directory in which the checkpoints will be saved:

.. code-block:: python

    from transformers import TrainingArguments

    training_args = TrainingArguments("test_trainer")

Then we can instantiate a :class:`~transformers.Trainer` like this:

.. code-block:: python

    from transformers import Trainer

    trainer = Trainer(
        model=model, args=training_args, train_dataset=small_train_dataset, eval_dataset=small_eval_dataset
    )

To fine-tune our model, we just need to call

.. code-block:: python

    trainer.train()

which will start a training that you can follow with a progress bar, which should take a couple of minutes to complete
(as long as you have access to a GPU). It won't actually tell you anything useful about how well (or badly) your model
is performing however as by default, there is no evaluation during training, and we didn't tell the
:class:`~transformers.Trainer` to compute any metrics. Let's have a look on how to do that now!

To have the :class:`~transformers.Trainer` compute and report metrics, we need to give it a :obj:`compute_metrics`
function that takes predictions and labels (grouped in a namedtuple called :class:`~transformers.EvalPrediction`) and
return a dictionary with string items (the metric names) and float values (the metric values).

The 🤗 Datasets library provides an easy way to get the common metrics used in NLP with the :obj:`load_metric` function.
here we simply use accuracy. Then we define the :obj:`compute_metrics` function that just convert logits to predictions
(remember that all 🤗 Transformers models return the logits) and feed them to :obj:`compute` method of this metric.

.. code-block:: python

    import numpy as np
    from datasets import load_metric

    metric = load_metric("accuracy")

    def compute_metrics(eval_pred):
        logits, labels = eval_pred
        predictions = np.argmax(logits, axis=-1)
        return metric.compute(predictions=predictions, references=labels)

The compute function needs to receive a tuple (with logits and labels) and has to return a dictionary with string keys
(the name of the metric) and float values. It will be called at the end of each evaluation phase on the whole arrays of
predictions/labels.

To check if this works on practice, let's create a new :class:`~transformers.Trainer` with our fine-tuned model:

.. code-block:: python

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=small_train_dataset,
        eval_dataset=small_eval_dataset,
        compute_metrics=compute_metrics,
    )
    trainer.evaluate()

which showed an accuracy of 87.5% in our case.

If you want to fine-tune your model and regularly report the evaluation metrics (for instance at the end of each
epoch), here is how you should define your training arguments:

.. code-block:: python

    from transformers import TrainingArguments

    training_args = TrainingArguments("test_trainer", evaluation_strategy="epoch")

See the documentation of :class:`~transformers.TrainingArguments` for more options.


.. _keras:

Fine-tuning with Keras
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <iframe width="560" height="315" src="https://www.youtube.com/embed/rnTGBy2ax1c" title="YouTube video player"
   frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope;
   picture-in-picture" allowfullscreen></iframe>

Models can also be trained natively in TensorFlow using the Keras API. First, let's define our model:

.. code-block:: python

    import tensorflow as tf
    from transformers import TFAutoModelForSequenceClassification

    model = TFAutoModelForSequenceClassification.from_pretrained("bert-base-cased", num_labels=2)

Then we will need to convert our datasets from before in standard :obj:`tf.data.Dataset`. Since we have fixed shapes,
it can easily be done like this. First we remove the `"text"` column from our datasets and set them in TensorFlow
format:

.. code-block:: python

    tf_train_dataset = small_train_dataset.remove_columns(["text"]).with_format("tensorflow")
    tf_eval_dataset = small_eval_dataset.remove_columns(["text"]).with_format("tensorflow")

Then we convert everything in big tensors and use the :obj:`tf.data.Dataset.from_tensor_slices` method:

.. code-block:: python

    train_features = {x: tf_train_dataset[x] for x in tokenizer.model_input_names}
    train_tf_dataset = tf.data.Dataset.from_tensor_slices((train_features, tf_train_dataset["label"]))
    train_tf_dataset = train_tf_dataset.shuffle(len(tf_train_dataset)).batch(8)

    eval_features = {x: tf_eval_dataset[x] for x in tokenizer.model_input_names}
    eval_tf_dataset = tf.data.Dataset.from_tensor_slices((eval_features, tf_eval_dataset["label"]))
    eval_tf_dataset = eval_tf_dataset.batch(8)

With this done, the model can then be compiled and trained as any Keras model:

.. code-block:: python

    model.compile(
        optimizer=tf.keras.optimizers.Adam(learning_rate=5e-5),
        loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
        metrics=tf.metrics.SparseCategoricalAccuracy(),
    )

    model.fit(train_tf_dataset, validation_data=eval_tf_dataset, epochs=3)

With the tight interoperability between TensorFlow and PyTorch models, you can even save the model and then reload it
as a PyTorch model (or vice-versa):

.. code-block:: python

    from transformers import AutoModelForSequenceClassification

    model.save_pretrained("my_imdb_model")
    pytorch_model = AutoModelForSequenceClassification.from_pretrained("my_imdb_model", from_tf=True)

.. _pytorch_native:

Fine-tuning in native PyTorch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <iframe width="560" height="315" src="https://www.youtube.com/embed/Dh9CL8fyG80" title="YouTube video player"
   frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope;
   picture-in-picture" allowfullscreen></iframe>

You might need to restart your notebook at this stage to free some memory, or execute the following code:

.. code-block:: python

    del model
    del pytorch_model
    del trainer
    torch.cuda.empty_cache()

Let's now see how to achieve the same results as in :ref:`trainer section <trainer>` in PyTorch. First we need to
define the dataloaders, which we will use to iterate over batches. We just need to apply a bit of post-processing to
our :obj:`tokenized_datasets` before doing that to:

- remove the columns corresponding to values the model does not expect (here the :obj:`"text"` column)
- rename the column :obj:`"label"` to :obj:`"labels"` (because the model expect the argument to be named :obj:`labels`)
- set the format of the datasets so they return PyTorch Tensors instead of lists.

Our `tokenized_datasets` has one method for each of those steps:

.. code-block:: python

    tokenized_datasets = tokenized_datasets.remove_columns(["text"])
    tokenized_datasets = tokenized_datasets.rename_column("label", "labels")
    tokenized_datasets.set_format("torch")

    small_train_dataset = tokenized_datasets["train"].shuffle(seed=42).select(range(1000))
    small_eval_dataset = tokenized_datasets["test"].shuffle(seed=42).select(range(1000))

Now that this is done, we can easily define our dataloaders:

.. code-block:: python

    from torch.utils.data import DataLoader

    train_dataloader = DataLoader(small_train_dataset, shuffle=True, batch_size=8)
    eval_dataloader = DataLoader(small_eval_dataset, batch_size=8)

Next, we define our model:

.. code-block:: python

    from transformers import AutoModelForSequenceClassification

    model = AutoModelForSequenceClassification.from_pretrained("bert-base-cased", num_labels=2)

We are almost ready to write our training loop, the only two things are missing are an optimizer and a learning rate
scheduler. The default optimizer used by the :class:`~transformers.Trainer` is :class:`~transformers.AdamW`:

.. code-block:: python

    from transformers import AdamW

    optimizer = AdamW(model.parameters(), lr=5e-5)

Finally, the learning rate scheduler used by default is just a linear decay from the maximum value (5e-5 here) to 0:

.. code-block:: python

    from transformers import get_scheduler

    num_epochs = 3
    num_training_steps = num_epochs * len(train_dataloader)
    lr_scheduler = get_scheduler(
        "linear",
        optimizer=optimizer,
        num_warmup_steps=0,
        num_training_steps=num_training_steps
    )

One last thing, we will want to use the GPU if we have access to one (otherwise training might take several hours
instead of a couple of minutes). To do this, we define a :obj:`device` we will put our model and our batches on.

.. code-block:: python

    import torch

    device = torch.device("cuda") if torch.cuda.is_available() else torch.device("cpu")
    model.to(device)

We now are ready to train! To get some sense of when it will be finished, we add a progress bar over our number of
training steps, using the `tqdm` library.

.. code-block:: python

    from tqdm.auto import tqdm

    progress_bar = tqdm(range(num_training_steps))

    model.train()
    for epoch in range(num_epochs):
        for batch in train_dataloader:
            batch = {k: v.to(device) for k, v in batch.items()}
            outputs = model(**batch)
            loss = outputs.loss
            loss.backward()

            optimizer.step()
            lr_scheduler.step()
            optimizer.zero_grad()
            progress_bar.update(1)

Note that if you are used to freezing the body of your pretrained model (like in computer vision) the above may seem a
bit strange, as we are directly fine-tuning the whole model without taking any precaution. It actually works better
this way for Transformers model (so this is not an oversight on our side). If you're not familiar with what "freezing
the body" of the model means, forget you read this paragraph.

Now to check the results, we need to write the evaluation loop. Like in the :ref:`trainer section <trainer>` we will
use a metric from the datasets library. Here we accumulate the predictions at each batch before computing the final
result when the loop is finished.

.. code-block:: python

    metric= load_metric("accuracy")
    model.eval()
    for batch in eval_dataloader:
        batch = {k: v.to(device) for k, v in batch.items()}
        with torch.no_grad():
            outputs = model(**batch)

        logits = outputs.logits
        predictions = torch.argmax(logits, dim=-1)
        metric.add_batch(predictions=predictions, references=batch["labels"])

    metric.compute()


.. _additional-resources:

Additional resources
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To look at more fine-tuning examples you can refer to:

- `🤗 Transformers Examples <https://github.com/huggingface/transformers/tree/master/examples>`__ which includes scripts
  to train on all common NLP tasks in PyTorch and TensorFlow.

- `🤗 Transformers Notebooks <notebooks>`__ which contains various notebooks and in particular one per task (look for
  the `how to finetune a model on xxx`).
