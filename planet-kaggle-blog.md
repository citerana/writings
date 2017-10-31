## The Planet Kaggle

![](imgs/banner.jpg)

### Amazon Basin Data

![](imgs/chipdesc.jpg)
<p align="center">source: <a href="https://www.kaggle.com/c/planet-understanding-the-amazon-from-space/data">Planet</a></p>

This past summer, Planet launched the *Understanding the Amazon from Space* Kaggle competition. We participated in this competition using Raster Vision, a system for analyzing aerial and satellite imagery using deep learning. Raster Vision works across several different tasks, including semantic segmentation, object detection and scene tagging as well as a range of datasets. The varied data within the Amazon competition gave us a challenging opportunity to add scene tagging functionality.

Planet provided over 100,000 chips from large images taken by a flock of satellites over the Amazon basin in 2016. These 40,000 training and 60,000 testing chips were given in both 3-band RGB JPEG and four band IR-RBG TIFF formats. The goal of tagging is to infer a set of labels for a given chip. For the Amazon images, there were 17 possible labels, which could be broadly split into three categories.

1. **atmospheric labels**: clear, partly cloudy, cloudy and hazy
2. **common labels**: primary, water, habitation, agriculture, road, cultivation and bare ground
3. **rare labels**: artisinal mine, blooming, blow down, conventional mine, selective logging and slash burn.

Each of the chips were labeled with ground truth labels through crowd-sourced labor. A well trained network can accurately predict the labels on a previously unseen chip.

<p align="center"><img src="imgs/label_hist.png" alt="Label histogram"/></p>
<p align="center">source: <a href="https://github.com/planetlabs/planet-amazon-deforestation/blob/master/planet_chip_examples.ipynb">anokas</a></p>

For this dataset, the labels were quite varied in frequency. Primary was by far the most common label, appearing in nearly 39,000 of the provided chips. Rare labels, on the other hand, were a source of concern. There were so few samples of rare labels, like conventional mining and blow down, that after splitting a portion of training data into a validation hold-out set, it was possible to have less than 100 data points for some rare labels versus the thousands of examples of images containing primary rainforest. Uncommon features can be very difficult to learn we expected that they would often be wrongly predicted by the model.

### Our Approach

Here are some examples of training chips with labels predicted by our best single-model network.

![Example tagging](imgs/debug_plots_labeled.png)
<p align="center">source: <a href="https://github.com/azavea/raster-vision/">Raster Vision</a></p>

In the above figure, the ground truth labels (ie. tagged by hand) for the Planet Kaggle dataset are bolded. Green bolded labels are correct. Unbolded and uncolored labels mean that they are false positives and have been incorrectly predicted for the chip. Red bolded labels are false negatives and missing from the network prediction. As with many Kaggle competitions involving image analysis, there are multiple stages in the prediction process that can be optimized for better accuracy. Improvements can be made in pre-processing the training data, switching or combining model architectures, adjusting optimizers, learning rates and augmenting the testing data. We placed 23rd overall out of nearly a thousand teams with a private leaderboard prediction F2 score of 0.93154 using the following techniques.

We normalized the provided images to be within a standardized range and used a range of image augmentations during the testing phase. We used a multi-architecture ensemble that took a majority vote over the predicted labels across 15 convolutional neural networks: (5) [ResNet50](https://arxiv.org/abs/1512.03385), (5) [Inception v3](https://arxiv.org/abs/1512.00567) and (5) [DenseNet121](https://arxiv.org/abs/1608.06993). These models were all trained using Adam as the optimizer, a decaying learning rate schedule and binary cross-entropy as the loss function.

The winning Kaggler, bestfitting, obtained a score of 0.93318 with their solution. More details on this approach can be found in bestfitting's [solution summary](http://blog.kaggle.com/2017/10/17/planet-understanding-the-amazon-from-space-1st-place-winners-interview/).

A point of note: submissions to the Kaggle competition were ranked using a potentially misleading version of an F2 score. The competition's scoring metric took the F2 score of each sample as the average rather than the F2 score for each label as the average. As a result, the F2 scoring metric tended to overlook mislabeling in rarer labels. Although the Amazon data simply asked for accurate labeling, we felt that large variation in appearances of certain labels was a key point in this goal. After all, a common application of scene tagging is to generate predictions to be further analyzed for any interesting or anomalous features. If we trained our model according to the F2 scoring metric, we may have obtained a slightly improved leaderboard score, but also increased the difficulty of identifying rarer labels. We can see how this may not be desirable behavior in a case where we may be more interested in the rare images with blooming flowers.

### Incorrect Labels

In order for this task to be performed accurately, it is crucial that the ground truth be genuinely truthful. In an ideal world, the criteria for a label is clear and distinct to humans and similarly obvious to trained neural networks. Unfortunately, Planet's crowd-labeled dataset had many ambiguous and, even worse, clearly incorrect labels. For example, we can see that the network often mistakes when to label a chip with the label `agriculture`. However, if we examine the ground truth labels for each chip, it's not obvious that the human classifications are correct either.

When we examined the data further, we found far more reason to be alarmed than simple human error. Many teams, including our own, attempted to train our models using exclusively TIFF chips. Although initial experiments with the JPEG images returned promising results, the data format contains only RGB, i.e. human-visible bands of light. The additional infrared band provided by the TIFF images is commonly used to calculate amounts of vegetation (NDVI) or water (NDWI) within an image. We had [previously](https://www.azavea.com/blog/2017/05/30/deep-learning-on-aerial-imagery/) used infrared, red and green bands to generate excellent results on [ISPRS Potsdam](http://www2.isprs.org/commissions/comm3/wg4/2d-sem-label-potsdam.html) data with Raster Vision's semantic segmentation task. As the Amazon data has significant amounts of both vegetation and water, it seemed obvious that the infrared band would be invaluable information.

However, after conducting a series of experiments using TIFF data we discovered that the results often drastically underperformed the same models using JPEG data. This bizarre behavior was likely the result of a couple factors.

First, there were differences in alignment between the TIFF and JPEG version of the same chip. When the chips were first created from the large satellite scenes, a portion of the chips were slightly misaligned between the two file versions. This misalignment didn't generally impact the correctness of the provided ground truth label but in rare cases, would render a label incorrect for one version of the chip. For example, cropping the leftmost 10% of an image might remove a river from the chip and therefore make the truth label of water incorrect for that chip.

Secondly, a small portion of the TIFF and JPEG chips were blatantly disimilar.

<p align="center"><img src="imgs/broken.jpg" alt="Bad chips" height=550px /></p>
<p align="center">source: <a href="https://www.kaggle.com/c/planet-understanding-the-amazon-from-space/discussion/32453">Heng CherKeng</a></p>

The left hand column displays the JPEG chip, while the right hand columns display the TIFF chip under two different visualizations. In each case, the labels indicate the chip has, among others, the `road` label. We can clearly see that in the TIFF chips were taken at some other time or location and clearly do not have a road within the image. If we teach a model that a image which does not have a road should be labeled road, this is a problem.

In almost all these cases, the chip that had the incorrect label was the TIFF chip. Our suspicion is that the crowd sourced labelers were by and large generating labels using only the JPEG chips. While the labelers were likely given both file versions for the chips, viewing the JPEG version of the chips is easier. The TIFF images required some normalization to make it visible to the human eye. As an unfortunate result, any discrepancies between the file versions favored the JPEG chips. Although the exact extent of misconfigured labels between the file types is unclear, dozens of such conflicts were easily spotted throughout the dataset. The resulting noise proved to be a major challenge in producing accurate results with even our most complex models using either TIFF files.

### Discussion

Many techniques that we tried did not increase our F2 score on the Amazon dataset. In several cases, we believe that the F2 score metric had unintentional effects on what constituted an accurate result. For example, we tried creating separate networks with different output layers for atmospheric labels versus the other labels. As there can be only one correct atmospheric label, it seemed that a softmax layer would output better predictions than a sigmoid layer. Although counterintuitive to our expectation, this did not yield improved results. The metric places greater value on predicions with both correct and incorrect labels more than predictions which do not have the correct label at all. We also had experiments that by oversampling rarer labels during training. Unfortunately, this did not improve our score. We suspect that the competition's method of calculating F2 score simply did not place a high value on the correct identification of the rarer labels. Even if our predictions were more accurate for these uncommon labels, it may have come at a cost to our overall F2 score across all the samples.

Sometimes it takes a clever idea or an unorthodox breakthrough to push the accuracy of a predictive model past all its competitors. In this particular Amazon Kaggle competition, vanilla machine learning solutions were the best performing.

Kaggle competitions are often won by very slim margins. In the case of the Amazon basin data, we spent a large portion of our development time trialing different methods of utilizing the TIFF images and postprocessing our predictions. Kaggle competitions incentize improving your position on the public leaderboard. The score for the public leaderboard was calculated using 66% of the testing chips. The winner, however, is determined by the final private leaderboard score. This score uses the remaining 34% of the testing data and requires that the model appropriately combat overfitting to the public testing data too much. Since the exact private leaderboard performance is unknown to all competitors until the competition's deadline, generalizing the model is crucial to a high Kaggle rank. Once you have a decent model, it takes great effort to make even small improvements. For the Amazon data, progress was further hampered by the unexpectedly noisy data. Of course, many techniques or tools that did not improve ourscore on the Amazon dataset remain useful for the scene tagging task and within Raster Vision.

### Future Work

The open-source project repository for Raster Vision can be found [here](https://github.com/azavea/raster-vision/). We are currently incorporating object detection into Raster Vision with a particular focus on strong performance for aerial and satellite imagery.
