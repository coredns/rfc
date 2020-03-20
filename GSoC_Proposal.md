# Anomaly Detection of CoreDNS Server through Machine Learning

| Status        | Proposed       |
:-------------- |:---------------------------------------------------- |
| **RFC #**     | [NNN](https://github.com/coredns/rfc/pull/NNN) (update when you have RFC PR #)|
| **Author(s)** | Chanakya Ekbote (@Chanakya-Ekbote) |
| **Sponsor**   |   |
| **Updated**   | YYYY-MM-DD                                           |
| **Obsoletes** |  |

## Objective

The aim of this project is to try and detect anomalies that occur in a CoreDNS server using a machine learning model developed in Keras. This project would help automise the process of anomaly detection, and reduce necessity to write anomaly detection 'rules'. 
What are we doing and why? What problem will this solve? What are the goals and
non-goals? This is your executive summary; keep it short, elaborate below.

## Motivation and Scope

To detect whether an anomaly has occured or not in a CoreDNS server, an engineer has to write specific 'rules'. If the data metrics follow those 'rules' they are classified as non - anomalies, however, if they violate these 'rules' they are classified as anomalies. Prometheus itself has these alerting rules. 

The processs of writing effective rules requires a lot of testing as well as experience in the field and moreover it may so happen that the engineer may not include some edge cases that may give false positives or false negatives.In addition to that, these anomaly detection rules are only for specific anomalies and hence may not generalize to other anomalies. Hence, it would make sense to model anomaly detection as a machine learning problem where the models could learn the relations betweem the metrics and the anomaly, which would then lead to a generalized anomaly detection solution. Moreover, this model can be continousuly updated based on new training data, which leads to adaptibility in the anomaly detection problem.

### Deliverables  

- At the end of the GSoC period, I would be be delivering a Keras Model that would detect anomalies.
- This model would be packaged as a part of the class. The class would contain various functions which would make it easy for anyone to retrain and modify the Keras Model, even those who have no idea about TensorFlow/Keras syntaxes
- At end of the project, we would have a huge collection of Prometheus data metrics which couls also be used for somone else to train other models as well as develop better alerting 'rules' 

Why this is a valuable problem to solve? Why is it related to CoreDNS?
Can this be done outside the scope of CoreDNS?

Which place will this proposal live in? A default plugin in [CoreDNS repo](https://github.com/coredns/coredns),
a separate repo in [CoreDNS org](https://github.com/coredns), or a external repo maintained outside of CoreDNS org?
Note exteranl repo maintained outside of CoreDNS org does not need a RFC.

## Design Proposal

To train a machine learning model, the main ingredient is data. The following methods allow us to garner data that could be used to train our machine learning model:

### Data Collection 
---
There are three methods through which we could collect data

- __Simulation of Normal Traffic__: We could use a simulation software that simulate Kubernets clusters to gee data metrics that correspond to non - anomalous data. The sofware that we can use is:
  - [k8s-cluster-simulator](https://github.com/pfnet-research/k8s-cluster-simulator) 
  
- __Get Acces to a Production Server__: We collect data metrics from a procution server, and use that to train the Keras Model.

The data would be split as follows. 80% would be used for training and the other 20% for validation. 

### Machine Learning Approaches

---

### Convolutional Autoencoder based Anomaly Detection

In this approach we'd be assuming that the data metrics we collect from both, the simulation as well as the production server are devoid of anomalies.

A Convolutional Autoencoder is a mahcine learning model that is useful for compressing the information contained in the input data to a lower dimenaional latent vector (of much lower dimensions.) Such a structure is usefull, as it helps discard the unecessary information present in the input data. Moreover, it is easier to derive inferences regarding the input data while working with learned representations of lower dimensions as it captures only the information that is relevant for reconstructing the data.  

<p float="left" align = "center">
  <img src="https://miro.medium.com/max/1700/1*I5MVGIrROrAnD3U_2Jm1Ng.png" width="600"/>
</p>

The encoder is tasked with compressing the information into a low dimenaion. The decoder makes sure that the latent vector correctly captures the information regarding the inpit data. 

The loss function for the entire model is the mean squared error between the inout and the output. The los function is as follows: 

<p align = "center" ><a href="https://www.codecogs.com/eqnedit.php?latex=Loss_{AE}&space;=&space;\frac{1}{m}\sum_{i&space;=1}^{m}(F(X_i)&space;-&space;X_i)^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Loss_{AE}&space;=&space;\frac{1}{m}\sum_{i&space;=1}^{m}(F(X_i)&space;-&space;X_i)^2" title="Loss_{AE} = \frac{1}{m}\sum_{i =1}^{m}(F(X_i) - X_i)^2" /></a></p>

Where, _F(X<sub>i</sub>)_ represnts the output of the Autoencoder, when the input data is _X<sub>i</sub>_.

Since the data metrics are time dependant, a small modification would have to be done, to the input 'image' of the Convolutional Autoencoder. In our case the time dependant metrics would be stacked up as a FIFO stack and used as the input to the Autoencoder. The input to the FIFO stack are the latest data mertrics. The size of the FIFO stack is a hyperparamter and can be changed depending on the results obtained during training.

__Inferences from the Model__

Once the Autoencoder has been trained, the model will be tested on anomalous data metrics (these can be filtered out based on the 'rules' described in Prometheus). An _L<sub>2</sub>_ norm will be calcualted between the mean of the latent representation of the anomaly free data metrics and the anomalous data metrics. The lowest _L<sub>2</sub>_ norm will be considered a threshold, and any latent vector whose distance from the mean that is greater than the threshold is cnosidered to be dereived from anomalous data. 


__Advantages__

Such a method can be used to find patterns between different anomaly types. This can be done by using a K-Means Clustering Algorithm or SVM's

__Disadvantage__

The optimization function doesent reflecrt the fact that anomalous and non anomalous data has to be seperated out. 

---

### A Convolutional Classfication Model

This would be a simple Convolutional Classification Model that predicts whetger the data metric corresponds to an anomaly or not. The model would basically be a binary classifier that predicts whether the data metrics coorespond to an anomaly ot not. The data that would be used is data from the production server as well as the simulation. 

Similar to the previous model, since the data metrics are time dependant, a small modification would have to be done, to the input 'image' of the Convolutional Autoencoder. In our case the time dependant metrics would be stacked up as a FIFO stack and used as the input to the Autoencoder. The input to the FIFO stack are the latest data mertrics. The size of the FIFO stack is a hyperparamter and can be changed depending on the results obtained during training.

The model will look something similar to this:
<p float="left" align = "center">
  <img src="http://www.wildml.com/wp-content/uploads/2015/11/Screen-Shot-2015-11-06-at-12.05.40-PM.png" width="600"/>
</p>

The loss function for the entire model would be the binary cross entropy loss. The loss function is as follows: 

<p align = "center" ><a href="https://www.codecogs.com/eqnedit.php?latex=Loss_{BC}&space;=&space;-&space;\frac{1}{m}&space;\sum_{i&space;=1}^{m}Y_{label_i}log(Y_{pred_i})&space;&plus;&space;(1&space;-&space;Y_{label_i})log(1&space;-&space;Y_{pred_i})" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Loss_{BC}&space;=&space;-&space;\frac{1}{m}&space;\sum_{i&space;=1}^{m}Y_{label_i}log(Y_{pred_i})&space;&plus;&space;(1&space;-&space;Y_{label_i})log(1&space;-&space;Y_{pred_i})" title="Loss_{BC} = - \frac{1}{m} \sum_{i =1}^{m}Y_{label_i}log(Y_{pred_i}) + (1 - Y_{label_i})log(1 - Y_{pred_i})" /></a></p>

Here, _Y_{label}_ gives the actual class of the data (whether it is an anomaly or not), and _Y_{pred}_ is the predicted label (whatthe model classifies the data as).

__Advantages__

The optimization function is directly corelated to our objective. 

__Disadvantages__

We won't be able to pinpoint a particualr type of anomaly 

---

### A Siameses based Classification Model

The main objective of this model is to make sure that the distance between the  latent vectors of non anomalous data is as small as possible, the distance between latent vectors of anomalous data is as small as possible, and also that the distance between anomalous and non - anomalous data is as large as possible, which in turn creates a cluster of anomalous data metrics in latent space that is far apart from the cluster of non-anomlaous data mterics in latent space, 

Similar o the previous two models, the data that would be used is data from the production server as well as the simulation. Moreover, since the data metrics are time dependant, a small modification would have to be done, to the input 'image' of the Convolutional Autoencoder. In our case the time dependant metrics would be stacked up as a FIFO stack and used as the input to the Autoencoder. The input to the FIFO stack are the latest data mertrics. The size of the FIFO stack is a hyperparamter and can be changed depending on the results obtained during training.

The model will be similar to this:

<p float="left" align = "center">
  <img src="https://miro.medium.com/max/960/1*g-561DsAfbU6gcVEk9AC4g.jpeg" width="600"/>
</p>


In a siamese network a pair of iamges are fed into the same network. If the images belong to the same class, then the distance is minimised. Else the distance is maximised.

The loss function is as follows: 

<p align = "center" ><a href="https://www.codecogs.com/eqnedit.php?latex=Loss_{SN}&space;=&space;\frac{1}{m}&space;\sum_{i&space;=1}^{m}Y_{label_i}(F(X_i)&space;-&space;X_i)^2&space;-&space;(1&space;-&space;Y_{label_i})(F(X_i)&space;-&space;X_i)^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Loss_{SN}&space;=&space;\frac{1}{m}&space;\sum_{i&space;=1}^{m}Y_{label_i}(F(X_i)&space;-&space;X_i)^2&space;-&space;(1&space;-&space;Y_{label_i})(F(X_i)&space;-&space;X_i)^2" title="Loss_{SN} = \frac{1}{m} \sum_{i =1}^{m}Y_{label_i}(F(X_i) - X_i)^2 - (1 - Y_{label_i})(F(X_i) - X_i)^2" /></a></p>

This loss function would minimise the distance if _Y<sub>i</sub>_ is equal to 1 (data is of the same class) or maximise the distance if _Y<sub>i</sub>_ is equal to 0 (the data is of different classes)  

__Inferences from the Model__

To understand whether the data metrics correspond to anomalous or non anomalous data, we cwill take the _L<sub>2</sub>_ norm between the latent vector rperestation of the data metric and the mean of the latent representation of the anomaly free data metrics as well as the mean of the latent representation of the anomalous data metrics. Whichever distance is smaller, that would be the class that the data metric is classified as. 

__Advantages__

The optimization function is directly corelated to our objective

__Disadvantages__

We won't be able to pinpoint a particualr type of anomaly 

All three of these models would be trained and then depnding on the results of the evaluation paramtersr of the model, one of them would be decided.

## Evalutaion Parameters

For each of these models the following metrics would be claculated:

- __Confusion Matrix__ : This is the matrix that gives an idea about the number of True Positives, True Negatives, False Positives and False Negatives.

- __PRescision__ : Precision is a metric that quantifies the number of correct positive predictions made. Therefore, Precision = TruePositives / (TruePositives + FalsePositives)

- __Recall__ : Recall is a metric that quantifies the number of correct positive predictions made out of all positive predictions that could have been made. Therefore, Recall = TruePositives / (TruePositives + FalseNegatives)

- __F Measure__ : F-Measure provides a way to combine both precision and recall into a single measure that captures both properties. Therefore, F-Measure = (2 * Precision * Recall) / (Precision + Recall) 

- __REsults on testing data__ : Some classes (based on the rules) will not be used for training or validation. They will purely be used for testing purposes to know how the model would respond to never been seen before anomalous data.

Depending on these results the models would be retrained, and later on a single model would be shorlisted from the three as a deliverable.

## Retraining the Model

Retraining the model will be of prime importance, as if the engineer pbserves some anomalous but the model predicts that no anomaly occured. In that case, the engineer labels this data and then the model is retrained. This is applicalbe  only for models 2 and 3. 

## Packaging it as a Class

The trained model would be packaged as a class. Moreover the classes would contain functions (given below) that would make training and testing the model much easier, even if he/she has no prior knowledge about TensorFlow/Keras syntaxes. The functions are as follows: 

```python
def import_weights()
   """
     Imports the trained weights
   """
```

```python
def initalize_new_weights()
   """
     Initializes new weights to the network
   """
```

```python
 def train(number_of_iterations, batch_size, data)
   """
     Trains the network depending on certain hyperparameters
   """ 
   # Args:
   # number_of_iterations: determines the number of iterations 
   # the model trains for
   # batch_size: determines the batch size
   # data: the data on which the model trains
```

```python
 def predcit(data)
   """
     Predicts the class of the data
   """ 
   # Args:
   # data: the data on which the model has to predict
```

More functions can be added on further discussion, 

## Timeline

**Before 27 April:**  

I will go through the required DNS concepts as well as the essential features of Prometheus.

**27 April to 18 May:** 

I will communicate with the CoreDNS community and particiape in any intersting doscussion that comes up. I will further finialize the ideas regarding the project by corresponding with the mentors as well as the members of the community. 


**18 May to 31st May:** 

In this period, data collection will be main priority. Data would be collected from the simulation software. Moreover, a production server would be created (if its not readily avilable), and the data would be collected from the same. In addition to that, the data will be filtered and preprocessed for training on the three machine learning models

**1st June to 7th June:** 

The first model wil be trained and the training for the same will begin. 

**8th June to 14th June:** 

The fsecond model wil be trained and the training for the same will begin. 

**15th June to 21st June:** 

The third model wil be trained and the training for the same will begin. 

**22nd July to 28th July:** 

The results are compiled in this section and discussed further with the mentors. The models are then retrained.

**29th July to 10th August:** 

The results are again compiled and then a model is shorlisted. The model is then packaged as a class and documentation is provided for the same. 


**10th to 17th August:** 

Period to account for any unforseen cirmumstances. 


## Questions and Discussion Topics

Seed this with open questions you require feedback on from the RFC process.

