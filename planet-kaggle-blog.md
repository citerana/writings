## Scene Tagging and the Planet Kaggle Competition

### The Task

![](imgs/tag_correlation.png#center)
![](imgs/tags.png#center)
*source: [anokas](https://www.kaggle.com/anokas/data-exploration-analysis)*

Raster Vision is a system for analyzing aerial and satellite imagery using deep learning and works across several different tasks and datasets. These tasks include semantic segmentation, object detection and scene tagging. As part of building a robust tagging component, we competed in Planet's Understanding the Amazon from Space Kaggle competition.

The competition provided approximately 40,000 train and 60,000 test images in both 3 band JPEG and four band TIFF formats taking by a flock of satellites over the year of 2016. Each of the chips were labeled with the ground truth tags by hand, primarily through crowd-sourced labor. This meant that the dataset had noticeable amounts of ambiguous and clearly incorrect labels for both sets of images. The resulting noise proved to be an additional challenge in producing accurate results with the Amazon data.

![](imgs/chipdesc.jpg)
*source: [Planet](https://www.kaggle.com/c/planet-understanding-the-amazon-from-space/data)*

The goal of tagging is to infer a set of labels for each image. For this competition, there were 17 possible tags, broadly split into three categories. 1) atmospheric labels: clear, partly cloudy, cloudy and hazy 2) common labels: primary, water, habitation, agriculture, road, cultivation and bare ground 3) rare labels: artisinal mine, blooming, blow down, conventional mine, selective logging and slash burn.

In order for this task to perform accurately, it is important that the labeling criteria be distinct and unambiguous in the training data. For example, we can see that the network often mistakes when to label a chip with the tag `agriculture`. However, if we examine the ground truth tags for each chip, it's not obvious that the human classifications are correct either.

-data, input/submission format, deadline

###Immediate Plan of Attack

-briefly, data generation/pre and post processing techniques
-the purpose of rastervision with usage of conv neural nets + ensembling
-avoiding overfitting

###Issues with Misconfigured Labels on .tif vs .jpg

-discuss problem of ambiguous/inconsistent labeling
-variation between tif and jpg, therefore unavailability of IR band
	-can discuss prior success with IRRG here

###Best Results

Here are some examples of training chips with labels predicted by our best single-model network.

![Example tagging](imgs/debug_plots_labeled.png)

In the above figure, the ground truth tags (ie. tagged by hand) for the Planet Kaggle dataset are bolded. Green bolded tags are correct. Unbolded and uncolored tags mean that they have been incorrectly predicted for the chip. Red bolded tags are ones missed by the network prediction.

In this competition, we placed 23nd overall out of nearly a thousand teams with a private leaderboard prediction accuracy of 93.154% using the following techniques.

The winning Kaggler, bestfitting, obtained a score of 93.318% accuracy with their solution.

More details on this approach can be found in bestfitting's [solution summary](https://www.kaggle.com/c/planet-understanding-the-amazon-from-space/discussion/36809).

* [DenseNet121](https://arxiv.org/abs/1608.06993)

-single model: DenseNet121 with cyclic?
-multiple model: prediction majority vote over 5x5x5x (incep, resnet, dense)?

show scores of the winner, talk a bit about dehazing, don't know how much it helps

###Struggles/Less Successful Results

#### Other model architectures
* [ResNet50](https://arxiv.org/abs/1512.03385)
* [Inception v3](https://arxiv.org/abs/1512.00567)
* [DenseNet121](https://arxiv.org/abs/1608.06993)
* [DenseNet169](https://arxiv.org/abs/1608.06993)
-resnet50 (good tester network)
-fcn
-wrn
-test-time/image time augmentation
-image zooming (lossless vs lossy transformations)
-talk a bit about how these models differ

-creating separate networks for softmax versus sigmoid, one hot labeling
-boosting presence of rarer labels

-identifying best optimizer:
	-sgd? Adam? cyclic? Yellowfin?
	- is hand tuning worth

Kaggle competitions are often won by very slim margins. In the case of the Amazon rainforest data, we spent a significant amount of development time trialing different methods of preprocessing and postprocessing the images .

Beating random predictor

### Discussion

-f2 score didn't incentivize the right behavior
-some competitions are won by clever ideas but simple, vanilla was just the best
-5 points of we tried this method --> and this was the score relationship
		- create a graphic describing this

###Future Work

The repository for Raster Vision can be found [here](https://github.com/azavea/raster-vision/) and is open to the public.

Nature of kaggle competitions, improving results against private leaderboard means not overfitting/aka generalization results through creating many models, identifying accurate but disjoint predictions and then averaging these results together.
