# Model configuration

The configuration process for DyGIE relies on the `jsonnet`-based configuration system for [AllenNLP](https://guide.allennlp.org/using-config-files). For more information on the AllenNLP configuration process in general, take a look at the AllenNLP [guide](https://guide.allennlp.org).

DyGIE adds one (regrettable but unavoidable) layer of complexity on top of this. It factors the configuration into:

- Components that are common to all DyGIE models. These are defined in [training_config/template.libsonnet].
- Components that are specific to single model trained on a particular dataset. These are contained in the `jsonnet` files in [training_config]. They use the jsonnet inheritance mechanism to extend the base class defined in `template.libsonnet`.  For more on jsonnet inheritance, see the [jsonnet tutorial](https://jsonnet.org/learning/tutorial.html)


## Required settings

The [training_config/template.libsonnet] file leaves three variables unset. These must be set by the inheriting object. For an example of how this works, see [training_config/scierc_lightweight.jsonnet].

- `data_paths`: A dict with paths to the train, validation, and test sets.
- `loss_weights`: Since DyGIE has a multitask objective, the individual losses are combined based on user-determined loss weights.
- `target_task`: After each epoch, the AllenNLP trainer assesses dev set performance, and saves the model state that achieved the highest performance. Since DyGIE is multitask, the user must specify which task to use as the evaluation target. The options are [`ner`, `rel`, `coref`, and `events`].

Note that if you create your own config outside of the `training_config` directory, you'll need to modify the line
```jsonnet
local template = import "template.libsonnet";
```
so that it points to the template file.


## Optional settings

The user may also specify:

- `bert_model`: The name of a pretrained BERT model available on [HuggingFace Transformers](https://huggingface.co/transformers/). The default is `bert-base-cased`.
- `max_span_width`: The maximum span length enumerated by the model. In pratice, 8 performs well.
- `cuda_device`: By default, training is performed on CPU. To train on a GPU, specify a device.


## Changing arbitrary parts of the template

The jsonnet object inheritance model allows you to modify any (perhaps deeply-nested) field of the base object using `+:` notation; see the jsonnet docs for more detail on this. For example, if you'd like to change the batch size and the learning rate on the optimizer, you could do:

```jsonnet
template.DyGIE {
  ...
  data_loader +: {
    batch_size: 5
  },
  trainer +: {
    optimizer +: {
      lr: 5e-4
    }
  }
}
```

You can also add additional fields to the base class. For instance, if you'd like to train a model using an existing vocabulary you could add
```jsonnet
template.DyGIE {
  ...
  vocabulary: {
    type: "from_files",
    directory: [path_to_vocab_files]
  }
}
```

## A full example

```jsonnet
local template = import "template.libsonnet";

template.DyGIE {
  // Required "hidden" fields.
  data_paths: {
    train: "data/scierc/processed_data/json/train.json",
    validation: "data/scierc/processed_data/json/dev.json",
    test: "data/scierc/processed_data/json/test.json",
  },
  loss_weights: {
    ner: 1.0,
    relation: 1.0,
    coref: 0.0,
    events: 0.0
  },
  target_task: "rel",

  // Optional "hidden" fields
  bert_model: "allenai/scibert_scivocab_cased",
  cuda_device: 0,
  max_span_width: 10,

  // Modify the data loader and trainer.
  data_loader +: {
    batch_size: 5
  },
  trainer +: {
    optimizer +: {
      lr: 5e-4
    }
  },

  // Specify an external vocabulary
  vocabulary: {
    type: "from_files",
    directory: "vocab"
  },
}
```