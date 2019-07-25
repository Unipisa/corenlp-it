# CoreNLP-it
A collection of CoreNLP add-on modules and models for processing Italian texts developed by the [CoLing Lab Team](http://colinglab.humnet.unipi.it/) of [University of Pisa](https://www.unipi.it/).

The CoreNLP-it package provides a collection fo classes and models that are built as an add-on to the Stanford CoreNLP.\
The main purpose of CoreNLP-it is to exploit the CoreNLP framework in order to deal with Italian texts and produce an output that is fully compliant with the *Universal Dependencies(UD)* guidelines for representation.
In particular, the system can deal with multi-word tokens that are often found in Italian text, producing a **CoNLL-U output**.
For what concerns Italian, we built custom annotator classes to deal with ***tokenization***, ***sentence splitting***, ***lemmatization*** and ***Universal PoS tags***.

Given the way it is built, the system provides out-of-the-box capability to deal with texts in other languages provided with an UD Treebank. In particular, the system offers the possibility to create basic models for language dependant tasks (tokenization - sentence splitting, lemmatization) directly from an UD Treebank file.

## USAGE

CoreNLP-it hinges on the original CoreNLP structure, by keeping the original usage intact. In particular, three custom annotators have to be specified:

```bash 
customAnnotatorClass.statTokSent: it.unipi.fileli.colinglab.pipeline.stat_tok_sent.annotator.StatTokSentAnnotator

customAnnotatorClass.upos: it.unipi.fileli.colinglab.pipeline.upos.UPosAnnotator

customAnnotatorClass.udLemma: it.unipi.fileli.colinglab.pipeline.UD_Lemma.UDLemmaAnnotator
```

In addition, the ``CLASS PATH`` for the CoreNLP-it folder has to be specified (e.g. from the ``-cp`` argument for the command line usage).

Each custom annotator replaces entirely the original CoreNLP annotator, both for input and output.

- StatTokSent replaces tokenize and ssplit.
- udLemma replaces lemma
- upos adds universal PoS tags and requires annotations up to PoS tagging (i.e. it has to be called after pos annotator in the pipeline).

Each custom annotator requires its own specific properties.

We strongly suggest to use the properties file by following the [CoreNLP guidelines](https://stanfordnlp.github.io/CoreNLP/cmdline.html).

An example of command line usage with a properties file:

```bash
$ java -cp "<path-to-corenlp-directory>/*:<path-to-corenlp-it>/*" -Xmx4g edu.stanford.nlp.pipeline.StanfordCoreNLP -props <path-to-props-file>
```

## CUSTOM ANNOTATOR AND CLASSES
### StatTokSentAnnotator

``StatTokSentAnnotator`` performs tokenization and sentence splitting on raw texts.

Tokenization and Sentence Splitting are approached simultaneously, by using:

1) a *character-based statistical model*. The model is trained to recognize each character as either being the begin of a sentence/token (both single-word and multi-word), inside a token, or outside any token.

2) a *set of rules* to further split multi-word tokens in their respective components in order to improve tokenization accuracy.

#### Usage
The custom annotator has 3 properties:

- .**model** : path to the statistical model for tokenization
- .**multiWordRules** : path to the file containing additional rules for multi-word tokens
- .**windowSize** : used to define the window size for extracting features of each character in the text. The default model uses a windowSize = 4

The custom annotator produces the same output, in term of CoreAnnotations, as the original `tokenize` and `ssplit` annotators combined each other.

#### StatTokSent class

The custom annotator uses a `StatTokSent` object to perform tokenization and sentence splitting.\
The `StatTokSent` class performs tokenization on raw texts.\
The `StatTokSent` class can be called as a standalone (command line) tool to tokenize and split a document into sentences.\
In this case, the class can be called with three arguments:

- **textFile**[required]: path to the text file to tokenize.
- **model**[required] : path to the statistical model for tokenization.
- **multiWordRules**[not required] : path to the multi-word rules file.

At this point, the class simply prints to stdout a token-index per line, with sentences separated from a blank line. This function will definitely be improved in the future.

#### StatTokSentTrainer class

The package includes a utility class to train a new tokenizer-sentence splitter model.\
At the moment, the classifier is implemented by means of the ColumnDataClassifier provided in the Stanford CoreNLP framework.\
The model can be trained using a properties (.props) file. The properties file must contain both `ColumnDataClassifier` specifications and `StatTokSentTrainer` specific arguments.\
The default model is trained by using a window size of 4 (i.e. 4 characters before, 4 characters after) and the case (i.e. upper, lower) of the character itself. For this reason, 9 features have to be specified. Each feature must have the `.useString` flag set to `True`.\
In order to train with a different window size, both the number of features and the -windowSize argument of the trainer/annotator/class have to be updated accordingly.

Arguments specific to the `StatTokSentTrainer`  are the following:

- **windowSize**[required] : size of the window used for extracting the features. Must exactly match (# of features -1)/2.
- **trainFile**[required]  : path to a CoNLL-U formatted file to train the model.
- **multiWordRulesFile**[not required] : path to a multi-word rules file. Such rules will be used in the training phase in order to treat multi-word tokens as a single token, thus performing the split only using rules during the tagging phase.
- **inferMultiWordRules**[not required] : if set to 1, and no -multiWordRulesFile is specified, multi-word tokens are inferred (before training) directly from the CoNLL-U training file. (not required)
- **serializeTo**[required] : path to the produced model (required)

Note: we plan to update the `StatTokSentTrainer` class to include a testing function that can test the classifier against a CoNLL-U formatted file.

### UPosAnnotator

The `UPosAnnotator` simply serves the purpose of mapping *language specific PoS* tags (xPoS) into *Universal PoS tags* (uPoS) as specified by UD.
The `UPosAnnotator` provides an annotation for the `CoreAnnotations.CoarseTagAnnotation`.

The annotator requires an external file (included in the distribution for Italian, based on ISDT Treebank) where such mapping is specified.\
The mapping file format is the following: a pair of TAB-separated xPoS and uPoS per line.\
If the mapping is not provided, the UPosAnnotator simply duplicates the xPoS tag (`PartOfSpeechAnnotation`) onto the uPoS tag (`CoarseTagAnnotation`).

The annotator has a single argument:

- .**uPoS-mapping**[not required] : specifies the path to the file where the mapping is contained.

Note: The annotator must be called within the pipeline *after* the pos tagger annotator.

### UDLemmaAnnotator

The `UDLemmaAnnotator` provides both the `LemmaAnnotation` and the `CoNLLUFeats` annotations.\
The annotator relies on an external file containing mapping between forms and lemmas of words.\
In particular, such file must be formatted as follows:

- each line contains one form and one or more triples of (PoS, lemma, morphological features).
- each line is TAB-separated.

The `UDLemmaAnnotator` has two arguments:

- .**model**[required] : path to the mapping file, or to a UDVocabulary serialized object. 
- .**serializeTo**[not required] : output path to the serialized file. In the default configuration, the serialized vocabulary is written in the same directory as the original file.

The `UDLemmaAnnotator` exploit an `UDVocabulary` object to identify lemmas from (token, PoS) pairs.

#### UDVocabulary class

The `UDVocabulary` class contains methods to build a vocabulary from a text file, serialize it, search it for lemmas given a (token, PoS) pair.\
The `UDVocabulary` class can be used as a standalone tool to generate serialized vocabularies from files (experimental), and to test performances of a Vocabulary against a CoNLL-U Treebank. The test is performed by measuring accuracy of prediction of lemmas given (Token, PoS) pairs.

In order to use the `UDVocabulary` class as a standalone (command line) tool, the following arguments can be used:

- **vocabularyFile**[required] : path to either a serialized `UDVocabulary` object or a text file.
- **testFile**[not required] : path to a CoNLL-U formatted file for testing.
- **serializeTo**[not required] : output path to the serialized file.
- **serialize**[not required] : in case -serializeTo is not called, this flag allows the `UDVocabulary` object to be serialized in the same directory as the `-vocabularyFile`.


NB: serialization of the lemma vocabulary from the `UDVocabulary` class is still experimental.\
We strongly suggest to launch the CoreNLP pipeline and specify a text file and a `-serializeTo` path, in order to build a model that can be used later directly from the pipeline.

## ITALIAN MODELS

In the package, we provide mapping files for Universal PoS Tags and Lemmas for Italian and a jar file containing models for the statistical tokenization, PoS Tagging and Dependency Parsing.\
The default model for tokenization is trained on the collection of all UD available Treebanks for Italian.\
Models for the Stanford MaxEnt PoS Tagger and Neural Network Dependency Parser are trained on the ISDT treebank.

The Italian models are included in CoreNLP-it.models.jar

A sample properties file with all arguments for each custom annotator is included.

## CITATION
The software is described in:

Bondielli A., Passaro L. C., and A. Lenci. "CoreNLP-it: A UD pipeline for Italian based on Stanford CoreNLP". Proceedings of the Fifth Italian Conference on Computational Linguistics (CLiC-it 2018).

If you use our software, please cite:

```
@inproceedings{BondielliEtal-2018,
	title={CoreNLP-it: A UD pipeline for Italian based on Stanford CoreNLP},
	author={Bondielli, Alessandro and Passaro, Lucia C. and Lenci, Alessandro},
	booktitle={Fifth Italian Conference on Computational Linguistics (CLiC-it 2018)},
	pages={57--61},
	year={2018},
	url = {http://ceur-ws.org/Vol-2253/paper24.pdf},
	organization={Accademia University Press}
}
```
