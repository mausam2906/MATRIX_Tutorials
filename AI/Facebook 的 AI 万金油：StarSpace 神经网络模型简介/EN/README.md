## StarSpace

StarSpace is a general-purpose neural model for efficient learning of entity embeddings for solving a wide variety of problems:

- Learning word, sentence or document level embeddings.
- Information retrieval: ranking of sets of entities/documents or objects, e.g. ranking web documents.
- Text classification, or any other labeling task.
- Metric/similarity learning, e.g. learning sentence or document similarity.
- Content-based or Collaborative filtering-based Recommendation, e.g. recommending music or videos.
- Embedding graphs, e.g. multi-relational graphs such as Freebase.
- NEW Image classification, ranking or retrieval (e.g. by using existing ResNet features).


In the general case, it learns to represent objects of different types into a common vectorial embedding space, hence the star ('*', wildcard) and space in the name, and in that space compares them against each other. It learns to rank a set of entities/documents or objects given a query entity/document or object, which is not necessarily the same type as the items in the set.

See the [paper](https://arxiv.org/abs/1709.03856) for more details on how it works.


## News

- New license and patents: now StarSpace is under BSD license. Checkout [LICENSE](https://github.com/facebookresearch/StarSpace/blob/master/LICENSE.md) and [PATENTS](https://github.com/facebookresearch/StarSpace/blob/master/PATENTS) for details.
- We added support for real-valued input and label weights: checkout the [File Format](https://github.com/facebookresearch/StarSpace/#file-format) and [ImageSpace](https://github.com/facebookresearch/StarSpace/#imagespace-learning-image-and-label-embeddings) section for more details on how to use weights in input and label.





## Requirements

StarSpace builds on modern Mac OS and Linux distributions. Since it uses C++11 features, it requires a compiler with good C++11 support. These include :

(gcc-4.6.3 or newer) or (clang-3.3 or newer)
Compilation is carried out using a Makefile, so you will need to have a working make.

You need to install Boost library and specify the path of boost library in makefile in order to run StarSpace. Basically:

    $wget https://dl.bintray.com/boostorg/release/1.63.0/source/boost_1_63_0.zip
    $unzip boost_1_63_0.zip
    $sudo mv boost_1_63_0 /usr/local/bi


Optional: if one wishes to run the unit tests in src directory, google test is required and its path needs to be specified in 'TEST_INCLUDES' in the makefile.


## Building StarSpace

In order to build StarSpace, use the following:

    git clone https://github.com/facebookresearch/Starspace.git
    cd Starspace
    make

## File Format

StarSpace takes input files of the following format. Each line will be one input example, in the simplest case the input has k words, and each labels 1..r is a single word:

    word_1 word_2 ... word_k __label__1 ... __label__r

This file format is the same as in fastText. It assumes by default that labels are words that are prefixed by the string __label__, and the prefix string can be set by "-label" argument.

In order to learn the embeddings, do:

    $./starspace train -trainFile data.txt -model modelSaveFile

where data.txt is a training file containing utf-8 encoded text. At the end of optimization the program will save two files: model and modelSaveFile.tsv. modelSaveFile.tsv is a standard tsv format file containing the entity embedding vectors, one per line. modelSaveFile is a binary file containing the parameters of the model along with the dictionary and all hyper parameters. The binary file can be used later to compute entity embedding vectors or to run evaluation tasks.

In the more general case, each label also consists of words:

    word_1 word_2 ... word_k <tab> label_1_word_1 label_1_word_2 ... <tab> label_r_word_1 .. 

Embedding vectors will be learned for each word and label to group similar inputs and labels together.

In order to learn the embeddings in the more general case where each label consists of words, one needs to specify the -fileFormat flag to be 'labelDoc', as follows:

    $./starspace train -trainFile data.txt -model modelSaveFile -fileFormat labelDoc

We also extend the file format to support real-valued weights (in both input and label space) by setting argument "-useWeight" to true (default is false). If "-useWeight" is true, we support weights by the following format

    word_1:wt_1 word_2:wt_2 ... word_k:wt_k __label__1:lwt_1 ...    __label__r:lwt_r

e.g.,

    dog:0.1 cat:0.5 ...

The default weight is 1 for any word / label that does not contain weights.

## Training Mode

To explain how it works in different train modes, we call the input of a particular example as "LHS" (stands for left-hand-side) and the label as "RHS" (stands for right-hand-side). StarSpace supports the following training modes (the default is the first one):

trainMode = 0:

- Each example contains both input and labels.
- If fileFormat is 'fastText' then the labels are individuals features/words specified (e.g. with a prefix label, see file format above).
- Use case: classification tasks, see tagspace example below.
- If fileFormat is 'labelDoc' then the labels are bags of features, and one of those bags is selected (see file format, above).
- Use case: retrieval/search tasks, each example consists of a query followed by a set of relevant documents.

trainMode = 1:
- Each example contains a collection of labels. At training time, one label from the collection is randomly picked as the RHS, and the rest of the labels in the collection become the LHS.
- Use case: content-based or collaborative filtering-based recommendation, see pagespace example below.


trainMode = 2:
- Each example contains a collection of labels. At training time, one label from the collection is randomly picked as the LHS, and the rest of the labels in the collection become the RHS.
- Use case: learning a mapping from an object to a set of objects of which it is a part, e.g. sentence (from within document) to document.

trainMode = 3:
- Each example contains a collection of labels. At training time, two labels from the collection are randomly picked as the LHS and RHS.
- Use case: learn pairwise similarity from collections of similar objects, e.g. sentence similiarity.

trainMode = 4:
- Each example contains two labels. At training time, the first label from the collection will be picked as the LHS and the second label will be picked as the RHS.
- Use case: learning from multi-relational graphs.

trainMode = 5:
- Each example contains only input. At training time, it generates multiple training examples: each feature from input is picked as the RHS, and other features surronding it (up to distance ws) are picked as the LHS.
- Use case: learn word embeddings in unsupervised way.


## Example use cases

### TagSpace word / tag embeddings

Setting: Learning the mapping from a short text to relevant hashtags, e.g. as in this paper. This is a classical classification setting.

Model: the mapping learnt goes from bags of words to bags of tags, by learning an embedding of both. For instance, the input “restaurant has great food <\tab> #restaurant <\tab> #yum” will be translated into the following graph. (Nodes in the graph are entities for which embeddings will be learned, and edges in the graph are relationships between the entities).

![](https://user-gold-cdn.xitu.io/2018/2/14/1619267e5b2c272d?imageslim)

Input file format:

    restaurant has great food #yum #restaurant


Command:

    $./starspace train -trainFile input.txt -model tagspace -label '#'

Example scripts:

We apply the model to the problem of text classification on AG's News Topic Classification Dataset. Here our tags are news article categories, and we use the hits@1 metric to measure classification accuracy. This example script downloads the data and run StarSpace model on it under the examples directory:

    $bash examples/classification_ag_news.sh


## PageSpace user / page embeddings

Setting: On Facebook, users can fan (follow) public pages they're interested in. When a user fans a page, the user can receive all things the page posts on Facebook. We want to learn page embeddings based on users' fanning data, and use it to recommend users new pages they might be interested to fan (follow). This setting can be generalized to other recommendation problems: for instance, embedding and recommending movies to users based on movies watched in the past; embed and recommend restaurants to users based on the restaurants checked-in by users in the past, etc.

Model： Users are represented as the bag of pages that they follow (fan). That is, we do not learn a direct embedding of users, instead, each user will have an embedding which is the average embedding of pages fanned by the user. Pages are embedded directly (with a unique feature in the dictionary). This setup can work better in the case where the number of users is larger than the number of pages, and the number of pages fanned by each user is small on average (i.e. the edges between user and page is relatively sparse). It also generalizes to new users without retraining. However, the more traditional recommendation setting can also be used.

![](https://user-gold-cdn.xitu.io/2018/2/14/16192685d36b0062?imageslim)

Each user is represented by the bag-of-pages fanned by the user, and each training example is a single user.

Input file format:

    page_1 page_2 ... page_M

At training time, at each step for each example (user), one random page is selected as a label and the rest of bag of pages are selected as input. This can be achieved by setting flag -trainMode to 1.


Command:

    $./starspace train -trainFile input.txt -model pagespace -label 'page' -trainMode 1

Example scripts:

To provide an example script, we choose the Last.FM (http://www.lastfm.com) dataset from HectRec 2011 and model it similarly as in the PageSpace setting: user is represented by the bag-of-artitsts listened by the user.

    $bash examples/recomm_user_artists.sh

## DocSpace document recommendation

Setting: We want to embed and recommend web documents for users based on their historical likes/click data.

Model: Each document is represented by a bag-of-words of the document. Each user is represented as a (bag of) the documents that they liked/clicked in the past. At training time, at each step one random document is selected as the label and the rest of the bag of documents are selected as input.

![](https://user-gold-cdn.xitu.io/2018/2/14/1619268ba6ede311?imageslim)

Input file format:

    roger federer loses <tab> venus williams wins <tab> world series ended
    i love cats <tab> funny lolcat links <tab> how to be a petsitter  


Each line is a user, and each document (documents separated by tabs) are documents that they liked. So the first user likes sports, and the second is interested in pets in this case.


### Command:

    ./starspace train -trainFile input.txt -model docspace -trainMode 1 -fileFormat labelDoc

### GraphSpace: Link Prediction in Knowledge Bases

Setting: Learning the mapping between entities and relations in Freebase. In freebase, data comes in the format

    (head_entity, relation_type, tail_entity)

Performing link prediction can be formalized as filling in incomplete triples like


    (head_entity, relation_type, ?) or (?, relation_type, tail_entity)

Model: We learn the embeddings of all entities and relation types. For each realtion_type, we learn two embeddings: one for predicting tail_entity given head_entity, one for predicting head_entity given tail_entity.

![](https://user-gold-cdn.xitu.io/2018/2/14/16192696723931a9?imageslim)


Example scripts:

This example script downloads the Freebase15k data from here and runs the StarSpace model on it:

    $bash examples/multi_relation_example.sh


## SentenceSpace: Learning Sentence Embeddings

Setting: Learning the mapping between sentences. Given the embedding of one sentence, one can find semantically similar/relevant sentences.

Model: Each example is a collection of sentences which are semantically related. Two are picked at random using trainMode 3: one as the input and one as the label, other sentences are picked as random negatives. One easy way to obtain semantically related sentences without labeling is to consider all sentences in the same document are related, and then train on those documents.


![](https://user-gold-cdn.xitu.io/2018/2/14/1619269e53dd90bc?imageslim)


Example scripts:

This example script downloads data where each example is a set of sentences from the same Wikipedia page and runs the StarSpace model on it:

    $bash examples/wikipedia_sentence_matching.sh

To run the full experiment on Wikipedia Article Search presented in [this paper](https://arxiv.org/abs/1709.03856), use [this script](https://github.com/facebookresearch/Starspace/blob/master/examples/wikipedia_article_search_full.sh) (warning: it takes a long time to download data and train the model):


    $bash examples/wikipedia_sentence_matching_full.sh

## ArticleSpace: Learning Sentence and Article Embeddings

Setting: Learning the mapping between sentences and articles. Given the embedding of one sentence, one can find the most relevant articles.

Model: Each example is an article which contains multiple sentences. At training time, one sentence is picked at random as the input, the remaining sentences in the article becomes the label, other articles are picked as random negatives (trainMode 2).

Example scripts:

[This example script ](https://github.com/facebookresearch/Starspace/blob/master/examples/wikipedia_article_search.sh)downloads data where each example is a Wikipedia article and runs the StarSpace model on it:

    $bash examples/wikipedia_article_search.sh


To run the full experiment on Wikipedia Article Search presented in [this paper](https://arxiv.org/abs/1709.03856), use [this script](https://github.com/facebookresearch/Starspace/blob/master/examples/wikipedia_article_search_full.sh) (warning: it takes a long time to download data and train the model):


    $bash examples/wikipedia_article_search_full.sh


## ImageSpace: Learning Image and Label Embeddings

With the most recent update, StarSpace can also be used to learn joint embeddings with images and other entities. For instance, one can use ResNet features (the last layer of a pre-trained ResNet model) to represent an image, and embed images with other entities (words, hashtags, etc.). Just like other entities in Starspace, images can be either on the input or the label side, depending on your task.

Here we give an example using CIFAR-10 to illustrate how we train images with other entities (in this example, image class): we train a ResNeXt model on CIFAR-10 which achieves 96.34% accuracy on test dataset, and use the last layer of ResNeXt as the features for each image. We embed 10 image classes together with image features in the same space using StarSpace. For an example image from class 1 with last layer (0.8, 0.5, ..., 1.2), we convert it to the following format:

    d1:0.8  d2:0.5   ...    d1024:1.2   __label__1



After converting train and test examples of CIFAR-10 to the above format, we ran [this example script](https://github.com/facebookresearch/StarSpace/blob/master/examples/image_feature_example_cifar10.sh):


    $bash examples/image_feature_example_cifar10.sh


and achieved 96.40% accuracy on an average of 5 runs.

## Full Documentation of Parameters

    Run "starspace train ..." or "starspace test ..."

    The following arguments are mandatory for train: 
      -trainFile       training file path
      -model           output model file path

    The following arguments are mandatory for test: 
      -testFile        test file path
      -model           model file path

    The following arguments for the dictionary are optional:
      -minCount        minimal number of word occurences [1]
      -minCountLabel   minimal number of label occurences [1]
      -ngrams          max length of word ngram [1]
      -bucket          number of buckets [2000000]
      -label           labels prefix [__label__]. See file format section.

    The following arguments for training are optional:
      -initModel       if not empty, it loads a previously trained model in -initModel and carry on training.
      -trainMode       takes value in [0, 1, 2, 3, 4, 5], see Training Mode Section. [0]
      -fileFormat      currently support 'fastText' and 'labelDoc', see File Format Section. [fastText]
      -validationFile  validation file path
      -validationPatience    number of iterations of validation where does not improve before we stop training [10]
      -saveEveryEpoch  save intermediate models after each epoch [false]
      -saveTempModel   save intermediate models after each epoch with an unique name including epoch number [false]
      -lr              learning rate [0.01]
      -dim             size of embedding vectors [100]
      -epoch           number of epochs [5]
      -maxTrainTime    max train time (secs) [8640000]
      -negSearchLimit  number of negatives sampled [50]
      -maxNegSamples   max number of negatives in a batch update [10]
      -loss            loss function {hinge, softmax} [hinge]
      -margin          margin parameter in hinge loss. It's only effective if hinge loss is used. [0.05]
      -similarity      takes value in [cosine, dot]. Whether to use cosine or dot product as similarity function in  hinge loss.
                   It's only effective if hinge loss is used. [cosine]
      -p               normalization parameter: we normalize sum of embeddings by deviding Size^p, when p=1, it's equivalent to taking average of embeddings; when p=0, it's equivalent to taking sum of embeddings. [0.5]
      -adagrad         whether to use adagrad in training [1]
      -shareEmb        whether to use the same embedding matrix for LHS and RHS. [1]
      -ws              only used in trainMode 5, the size of the context window for word level training. [5]
      -dropoutLHS      dropout probability for LHS features. [0]
      -dropoutRHS      dropout probability for RHS features. [0]
      -initRandSd      initial values of embeddings are randomly generated from normal distribution with mean=0, standard deviation=initRandSd. [0.001]
      -trainWord       whether to train word level together with other tasks (for multi-tasking). [0]
      -wordWeight      if trainWord is true, wordWeight specifies example weight for word level training examples. [0.5]

    The following arguments for test are optional:
      -basedoc         file path for a set of labels to compare against true label. It is required when -fileFormat='labelDoc'.
                   In the case -fileFormat='fastText' and -basedoc is not provided, we compare true label with all other labels in the dictionary.
      -predictionFile  file path for save predictions. If not empty, top K predictions for each example will be saved.
      -K               if -predictionFile is not empty, top K predictions for each example will be saved.

    The following arguments are optional:
      -normalizeText   whether to run basic text preprocess for input files [0]
      -useWeight       whether input file contains weights [0]
      -verbose         verbosity level [0]
      -debug           whether it's in debug mode [0]
      -thread          number of threads [10]

Note: We use the same implementation of word n-grams for words as in fastText. When "-ngrams" is set to be larger than 1, a hashing map of size specified by the "-bucket" argument is used for n-grams; when "-ngrams" is set to 1, no hash map is used, and the dictionary contains all words within the minCount and minCountLabel constraints.

## Utility Functions

We also provide a few utility functions for StarSpace：

## Show Predictions for Queries
A simple way to check the quality of a trained embedding model is to inspect the predictions when typing in an input. To build and use this utility function, run the following commands:

    make query_predict
    ./query_predict <model> k [basedocs]

where "<model>" specifies a trained StarSpace model and the optional K specifies how many of the top predictions to show (top ranked first). "basedocs" points to the file of documents to rank, see also the argument of the same name in the starspace main above. If "basedocs" is not provided, the labels in the dictionary are used instead.

After loading the model, it reads a line of entities (can be either a single word or a sentence / document), and outputs the predictions.

## Nearest Neighbor Queries
Another simple way to check the quality of a trained embedding model is to inspect nearest neighbors of entities. To build and use this utility function, run the following commands:

    make query_nn
    ./query_nn <model> [k]


where "<model>" specifies a trained StarSpace model and the optional K (default value is 5) specifies how many nearest neighbors to search for.

After loading the model, it reads a line of entities (can be either a single word or a sentence / document), and output the nearest entities in embedding space.

## Print Ngrams
As the ngrams used in the model are not saved in tsv format, we also provide a separate function to output n-grams embeddings from the model. To use that, run the following commands:

    make print_ngrams
    ./print_ngrams <model>

where "<model>" specifies a trained StarSpace model with argument -ngrams > 1.


## Print Sentence / Document Embedding

Sometimes it is useful to print out sentence / document embeddings from a trained model. To use that, run the following commands:

    make embed_doc
    ./embed_doc <model> [filename]

where "<model>" specifies a trained StarSpace model. If filename is provided, it reads each sentence / document from file, line by line, and outputs vector embeddings accordingly. If the filename is not provided, it reads each sentence / document from stdin.

