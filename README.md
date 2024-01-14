# Detection of Settlements without Electricity

## Introduction
Access to electricity is a critical element in modern human life, leading to better educational outcomes for children1, improved health care2, and greater safety for women3. It is the goal of many governments4 and organizations5 to expand access to electricity to every human settlement, but it can be difficult to locate these communities, as they are by definition not connected to a larger network. In 2021, the Image Analysis and Data Fusion Technical Committee of the Institute of Electrical and Electronics Engineers (IEEE) Geoscience and Remote Sensing Society (GRSS), Hewlett Packard Enterprise, SolarAid, and Data Science Experts organized the IEEE GRSS Data Fusion Contest to extend research in automatic detection of human settlements lacking access to electricity using multimodal and multitemporal remote sensing data.  
The geographic focus of this project was the Dezda and Salima townships in Malawi. Malawi is a country in southern Africa about the size of Pennsylvania, with a population of 21 million people who live on plateaus, rolling plains, and rounded hills6. Most people live in rural areas and work in agriculture, and 51% are below the poverty line7. 17 million people in Malawi do not have access to electricity, with a total population electrification rate of 14% (54% of urban areas and 5.5% of rural areas). Hydroelectricity accounts for 82% of all their electric power, followed by 12% from fossil fuels and 3.2% from solar power8. The need for greater access to electricity is acute, and therefore the need for accurate data and knowledge of the scope of the problem is vital.
The data for this project is composed of 98 tiles of 800x800 pixels, each resampled to a Ground Sampling Distance (GSD) of 10m, and covering a 64km2 section of the Dezda and Salima townships. Each tile has 98 channels, corresponding to data from four satellite datasets. The channels from each satellite are as follows9:
-Sentinel-1 polarimetric SAR dataset
 -2 channels corresponding to intensity values for VV (vertical transmit, vertical receive) and VH (vertical transmit, horizontal receive) polarization of microwave radiation
 -Number of images per channel: 4
-Sentinel-2 multispectral dataset
 -12 channels of reflectance data covering Visible and Near-Infrared (VNIR) and Short-Wave Infrared (SWIR) ranges at a GSD of 10m, 20m, and 60m. The cirrus band 10 is omitted as it contains data at cloud level only.
 -Number of images per channel: 4
-Landsat 8 multispectral dataset
 -11 channels of reflectance data covering VNIR, SWIR, and Thermal Infrared Sensor (TIRS) at GSD of 30m and 100m, and a Panchromatic band of all visible colors at a GSD of 15m
 -Number of images per channel: 3
-The Suomi Visible Infrared Imaging Radiometer Suite (VIIRS) night time dataset
  -1 channel of the Day-Night Band (DNB) sensor of the VIIRS, including the global daily measurements of nocturnal VNIR light at a GSD of 750m.
 -Number of images per channel: 9
The ground truth tiles are constructed according to two binary categories, defined as follows: If there is a building in a patch of 500m2, this area is considered to have human settlement, and if a fraction of a patch of 500m2 is illuminated, this area is considered to have electricity. These area sizes correspond to a 50x50 pixel square of the given tiles, so the ground truth image contains 256 labels in a 16x16 grid, as visualized in Figure 1. The four classes are:
1. Human settlements without electricity
2. No human settlements; without electricity
3. Human settlements with electricity
4. No human settlements; with electricity

There is notable related research in this field - in fact, the use of night-light remote sensing began in the 1970s as a way to forecast weather10, but has expanded into a wide range of estimation and monitoring, including for electricity consumption11 and grid reliability12. In 2013 and 2014 two studies were published which found that electrified villages tend to be brighter than unelectrified villages13, even when electrified villages have no public streetlights14. The research on sensing electrification continues, for example with tests on thresholds and new data overlays15. Likewise, the research on identifying human settlements continues even within IEEE GRSS, as their 2023 contest was centered on large-scale fine-grained building classification for semantic urban reconstruction16.
## Method 
### Data Engineering 
The data came in 98 tiles of 98 channels each, all of size 800x800 pixels. The ground truth labels were 16x16 pixel images, each pixel corresponding to a 50x50 pixel area of the tile. Accordingly, the first order of business was to slice the images into manageable tensors. This is done by isolating sections of the images and stacking them as individual images. . This process is iterated over multiple tiles, resulting in a list of tensors for each tile. The tensors for each tile are then stacked again, creating a final tensor that represents all channels for that tile. Finally, the tensors for all tiles are concatenated along the first axis, resulting in a single tensor containing the image data for the entire dataset, for a 60-tile training dataset size of (15360, 98, 50,50). This represents each sub-divided portion of the tile, and each of its channels.
From here labels are loaded from corresponding image files, representing the ground truth information for each tile. The label images are flattened, and the values are adjusted to convert from a 1-4 scale to a 0-3 scale. The adjusted labels are stored in a NumPy array. The flattened label arrays for all tiles are concatenated into a single array, representing the labels for the entire dataset.
The dataset exhibits a pronounced class imbalance, which can be demonstrated by examining the training dataset. Class 2 is the majority class, representing 53.09% of the samples. Class 1 follows closely, comprising 41.1% of the dataset. In contrast, Class 3 and Class 4 are significantly underrepresented, constituting only 4.4% and 1.37% of the samples, respectively. The prevalence of Class 2, in particular, could introduce a bias in model predictions, as the model might prioritize the majority classes. This imbalance poses challenges, especially for the accurate recognition of minority classes. Evaluation metrics like accuracy may not offer a complete picture of the model's performance in such scenarios. Using evaluation metrics that account for imbalances, such as F1 score, can help ensure a more nuanced understanding of the model's effectiveness across all classes.
### Data Augmentation
In order to enhance the diversity of the training dataset and improve the generalization capability of our deep learning model, we conducted two distinct trials of data augmentation. The first trial, referred to as the "light" augmentation, involved applying horizontal flip and rotation operations with a low probability. This approach aimed to introduce subtle variations to the training images without significantly altering their content. Conversely, the second trial, denoted as the "heavy" augmentation, incorporated a more extensive set of transformations. This included both horizontal and vertical flips, rotations, and cropping, all applied with a higher probability. The "heavy" augmentation strategy was designed to introduce more substantial variations and simulate a broader range of potential real-world scenarios, encouraging the model to become more robust to diverse image transformations. These augmentation strategies were systematically implemented to investigate their impact on model performance and assess their effectiveness in mitigating overfitting while promoting improved generalization across different test scenarios.
### Dimension Reduction 
Another method implemented was reducing the amount of data used to train the model. Initially, we reduced the data used in training from four to two satellites, VIIRS and Sentinel-2. This first reduced dataset is referred to as the “2 satellite” dataset in subsequent sections. In further experiments, the satellite data was further split based on its expected classification association. The VIIRS night time dataset was selected for determining electrification, as electricity is most visible at night. We decided to additionally use the Sentinel-2 multispectral dataset for human settlement classification, as not only was that a successful route for prior work17, but it has the most precise data of all the channels, with a GSD of 10m. The second reduced dataset is referred to as the “satellite split” dataset and is used in the dual, split model described in later sections. 
## Model Explanations
#### Metric Comparison
Due to the class imbalance and the design of the contest from which this project originated, F1 score was used as the metric to determine the best performing model. Multi-class F1 score was used with a weighted average of individual class F1 scores to account for the class imbalance. 
ResNet and SENet Model Comparison
ResNet-1818 was chosen as the first model to test. This model was chosen as a starting point due to its pretraining on ImageNet, its reduced training time thanks to residual connections and its known ability to generalize well to different types of data. 

A Squeeze-and-Excitation Network (SENet-20)19 was chosen as the second model architecture for experiments. The squeeze step in this architecture uses pooling to aggregate feature maps and produce a channel descriptor. The excitation step captures channel-wise dependencies and uses them to rescale subsequent features20. This model is not pretrained. Because of the large number of channels in this dataset, this model was chosen for its ability to account for channel importance. This choice was also influenced by previous work9 in which competitors in this contest showed best performance with the SENet model. 

The ResNet and SENet model architectures were updated to take in the appropriate number of channels from the input images, 98 channels for the full dataset, 57 channels for the 2 satellite truncated dataset, and 9 and 48 channels respectively for the dual split model architecture, each representing one satellite dataset. The final layers in each were also updated for classifying into the four categories. 

Each of these models was tuned on hyperparameters including batch size, learning rate and momentum. Batch sizes between 16 and 96 were tested. Learning rate spanned 0.005 to 0.03 and momentum ranged from 0.5 to 0.9. The best performing model (architecture and parameters) based on F1 score was chosen as the baseline comparison for further experiments. 

Due to time and resource constraints, since we wanted to test as many parameters and model variations as possible, we used the smallest versions of these models for comparison. 
### Architecture Additions
The next experiment involved adding layers to the best performing model from the first stage, this being the SENet model. Five different architecture additions were created and tested sequentially to determine effectiveness of individual parameters. These five additions are as follows: 
2 Linear layers added
5 convolutional layers added
5 convolutional layers added with pooling on 3 layers
5 convolutional layers added with pooling on 3 layers and batch normalization on 2 layers
10 convolutional layers added with pooling on 3 layers, and batch normalization on 3 layers

These parameters were chosen and implemented sequentially to determine individual effects of pooling and batch normalization as well as the effect of making the network deeper. 
Dual-Binary Model Architecture
Some work on this project topic incorporated splitting the model into two binary models, one each to classify human settlements and electricity. This method was also attempted in this project, and the datasets were modified to allow for a binary classification in each case. 

The best performing SENet architecture and parameters were used as the starting point for the dual-binary model experiments.This set of models were also hyperparameter-tuned on batch size and learning rate. 
### Satellite Splitting
Furthermore, as well as training two binary models on the 57 channels of two satellites, we also split the data into two groups by satellite. An electricity classifier was trained on the VIIRS night time dataset while a human settlement classifier was trained on the VNIR and SWIR data of the Sentinel-2 satellite.

## Experiments 
ResNet vs SENet (Hyperparameter Tuning)
Figure 2 shows the breakdown of hyperparameters tuned in this stage of experiments. The figure is colored by F1 score with the highest scores in red. The SENet-20 models performed better than ResNet-18 trials. Batch size had a significant effect on the training with a batch size of 16 outperforming all other batch sizes. The best learning rate was 0.03 and momentum was 0.9 resulting in an F1 score of 0.797.
Full Data vs 2 Sat Data
This set of experiments explored the performance improvement from reducing the data to the 2 satellite dataset. Figure 3 shows the hyperparameter tuning of the ResNet and SENet architectures on the 2 satellite dataset. The best performing model here is the SENet model on the 2 satellite data with the following parameters: batch size = 32,  learning rate = 0.03 and momentum = 0.9. This setup is used as the baseline for comparison with all further models and the starting point for hyperparameters for subsequent experiments. Even after further experimentation (as described in the following sections), this model proves to be the best performing model overall. We saw much improvement when reducing the amount of training data in this way: our best result was an F1 score of 0.797 on the full dataset, but 0.821 with just these two satellite datasets. In later sections, this best performing model will be referred to as the Baseline or the SENet 2sat model. 
Architecture Adjustments

Figure 4 shows the comparison of the SENet 2sat model (baseline) and the five architecture addition combinations. The architecture updates are ordered as described in the Methods section. This chart shows that the addition of any linear or convolutional layers causes performance degradation from the baseline. The final addition of the 10 convolutional layers performs better than the fourth setup which is 5 convolutional layers with pooling and batch normalization. The batch normalization has the biggest effect on performance as shown in the drop between architecture 3 and 4. 

### Data Augmentation
Figure 5 shows the results of the best performing models from each experiment including the best results from both “light” and “heavy” augmentation as described in the preceding section. The best F1 score was 0.8213 and 0.823 on the light and heavy augmentation respectively, making the heavy augmentation the best performing model among all models tested.
### Dual-Binary Model 
This set of experiments involved splitting the data into two binary sets, one for binary classification of human settlement and the other for electricity. Despite attempts at tuning the model, the human settlement models did not improve and stayed consistently worse than the SENet 2sat model. Due to dataset imbalance, the F1 score for the electricity training was 0.98 in the first epoch, but fell in the following epochs. The predictions from each of the binary models were combined back into the original four classes. The F1 score from this combination is included in Figure 5 for comparison with the other models. 
### Dual Satellite Split
Here, the goal was to take the satellites that were most relevant to the human settlements and electricity and use only those images to train the binary human and electricity models respectively. The results of the best performing human settlement model trained with only the VNIR and SWIR data of the Sentinel-2 satellite and the best performing electricity model trained on VIIRS nighttime dataset were combined in the same way as the dual binary model. Additionally, an augmented dataset was tested in this dual split model configuration. This combination result is shown in Figure 5. These approaches did not improve the F1 score over the SENet 2sat model. 
## Discussion 
There were several methods that caused model performance improvement over the course of our experiments. The first successful implementation was using SENet-20 over ResNet-18. The second was reducing the dimensionality of the data from four satellites to two. Lastly, implementing data augmentation showed improved classification results as well. 

Other methods implemented did not improve the F1 score of the models beyond the already listed successful implementations. These methods included adding layers to the model architecture and implementing dual-binary classification models for human settlements and electricity. A final attempt was made for improvement by splitting the satellite data that would be most impactful for human settlements and electricity and training the dual binary models on only those satellites. Unfortunately, these attempts did not lead to F1 score improvement. 

Our overall best performing model was the SENet-20 trained on the 2 satellite dataset with heavy augmentation, for a final F1 score of 0.823. 


## References
Akram, V. (2022). Causality between access to electricity and education: Evidence from BRICS countries. Energy RESEARCH LETTERS, 3(2). https://doi.org/10.46557/001c.32597
Cahill, B. (2021, March 22). Energy Access and Health Outcomes. Center for Strategic and International Studies. https://www.csis.org/analysis/energy-access-and-health-outcomes
Clark, L. (2021). Powering Households and Empowering Women: The Gendered Effects of Electrification in sub-Saharan Africa. Journal of Public and International Affairs. https://doi.org/https://jpia.princeton.edu/news/powering-households-and-empowering-women-gendered-effects-electrification-sub-saharan-africa
U.S.A.I.D. (2023, August 4). Sustainable development goal 7: Affordable and Clean Energy: Basic Page. U.S. Agency for International Development. https://www.usaid.gov/sdgs/sdg7
United Nations. (n.d.). Goal 7 | Department of Economic and Social Affairs. United Nations. https://sdgs.un.org/goals/goal7
Central Intelligence Agency. (n.d.). CIA Factbook: Malawi. Central Intelligence Agency. https://www.cia.gov/the-world-factbook/countries/malaw/#geography
Central Intelligence Agency. (n.d.). CIA Factbook: Malawi. Central Intelligence Agency. https://www.cia.gov/the-world-factbook/countries/malaw/#economy
Central Intelligence Agency. (n.d.). CIA Factbook: Malawi. Central Intelligence Agency. https://www.cia.gov/the-world-factbook/countries/malaw/#energy
IEEE GRSS. (2021, February 13). 2021 IEEE GRSS Data Fusion Contest Track DSE - GRSS-IEEE. GRSS. https://www.grss-ieee.org/community/technical-committees/2021-ieee-grss-data-fusion-contest-track-dse/
Levin, N.; Kyba, C.C.M.; Zhang, Q.L.; de Miguel, A.S.; Roman, M.O.; Li, X.; Portnov, B.A.; Molthan, A.L.; Jechow, A.; Miller, S.D.; et al. Remote sensing of night lights: A review and an outlook for the future. Remote Sens. Environ. 2020, 237, 111443.
Shi, K.F.; Yu, B.L.; Huang, Y.X.; Hu, Y.J.; Yin, B.; Chen, Z.Q.; Chen, L.J.; Wu, J.P. Evaluating the Ability of NPP-VIIRS Nighttime Light Data to Estimate the Gross Domestic Product and the Electric Power Consumption of China at Multiple Scales: A Comparison with DMSP-OLS Data. Remote Sens. 2014, 6, 1705–1724.
Shah, Z.; Taneja, J. Poster Abstract: Monitoring Electric Grid Reliability Using Satellite Data. In Proceedings of the 6th ACM International Conference on Systems for Energy-Efficient Buildings, Cities, and Transportation (BuildSys), New York, NY, USA, 13–14 November 2019; pp. 378–379.
Min, B.; Gaba, K.M.; Sarr, O.F.; Agalassou, A. Detection of rural electrification in Africa using DMSP-OLS night lights imagery. Int. J. Remote Sens. 2013, 34, 8118–8141.
Min, B.; Gaba, K.M. Tracking Electrification in Vietnam Using Nighttime Lights. Remote Sens. 2014, 6, 9511–9529.
Gao, X., Wu, M., Niu, Z., &amp; Chen, F. (2022). Global identification of unelectrified built-up areas by remote sensing. Remote Sensing, 14(8), 1941. https://doi.org/10.3390/rs14081941
Persello, C., Hänsch, R., Vivone, G., Chen, K., Yan, Z., Tang, D., Huang, H., Schmitt, M., &amp; Sun, X. (2023). 2023 IEEE GRSS Data Fusion Contest: Large-scale fine-grained building classification for Semantic Urban Reconstruction [technical committees]. IEEE Geoscience and Remote Sensing Magazine, 11(1), 94–97. https://doi.org/10.1109/mgrs.2023.3240233
Ma, Y., Li, Y., Feng, K., Xia, Y., Huang, Q., Zhang, H., Prieur, C., Licciardi, G., Malha, H., Chanussot, J., Ghamisi, P., Hansch, R., & Yokoya, N. (2021). The outcome of the 2021 IEEE GRSS Data Fusion Contest - track DSE: Detection of settlements without electricity. IEEE Journal of Selected Topics in Applied Earth Observations and Remote Sensing, 14, 12375–12385. https://doi.org/10.1109/jstars.2021.3130446
He, K., Zhang, X., Ren, S., & Sun, J. (2016). Deep residual learning for image recognition. 2016 IEEE Conference on Computer Vision and Pattern Recognition (CVPR). https://doi.org/10.1109/cvpr.2016.90
Hataya, R. (n.d.). Moskomule/SENet.pytorch: Pytorch implementation of Senet. GitHub. https://github.com/moskomule/SENet.pytorch
Ramadoss, V. (2022, November 1). Squeeze-and-excitation networks. Medium. https://medium.com/@Vinoth-Ramadoss/squeeze-and-excitation-networks-84e3db0e04e2
