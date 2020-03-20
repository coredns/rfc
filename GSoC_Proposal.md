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


### Machine Learning Approaches


#### Convolutional Autoencoder based Anomaly Detection

In this approach we'd be assuming that the data metrics we collect from both, the simulation as well as the production server are devoid of anomalies.

A Convolutional Autoencoder is a mahcine learning model that is useful for compressing the information contained in the input data to a lower dimenaional latent vector (of much lower dimensions.) Such a structure is usefull, as it helps discard the unecessary information present in the input data. Moreover, it is easier to derive inferences regarding the input data while working with learned representations of lower dimensions as it captures only the information that is relevant for reconstructing the data.  

<p float="left" align = "center">
  <img src="https://miro.medium.com/max/1700/1*I5MVGIrROrAnD3U_2Jm1Ng.png" width="600"/>
</p>

The encoder is tasked with compressing the information into a low dimenaion. The decoder makes sure that the latent vector correctly captures the information regarding the inpit data. 

The loss function for the entire model is the mean squared error between the inout and the output. The los function is as follows: 

<p align = "center" ><a href="https://www.codecogs.com/eqnedit.php?latex=Loss_{AE}&space;=&space;\frac{1}{m}\sum_{i&space;=1}^{m}(F(X_i)&space;-&space;X_i)^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Loss_{AE}&space;=&space;\frac{1}{m}\sum_{i&space;=1}^{m}(F(X_i)&space;-&space;X_i)^2" title="Loss_{AE} = \frac{1}{m}\sum_{i =1}^{m}(F(X_i) - X_i)^2" /></a></p>



Since the data metrics are time dependant, a small modification would have to be done, to the input 'image' of the Convolutional Autoencoder. The input to the C

This is the meat of the document, where you explain your proposal. If you have
multiple alternatives, be sure to use sub-sections for better separation of the
idea, and list pros/cons to each approach. If there are alternatives that you
have eliminated, you should also list those here, and explain why you believe
your chosen approach is superior.

Make sure you’ve thought through and addressed the following sections. If a section is not relevant to your specific proposal, please explain why, e.g. your RFC addresses a convention or process, not an API.


## Alternatives Considered
* Make sure to discuss the relative merits of alternatives to your proposal.

## Questions and Discussion Topics

Seed this with open questions you require feedback on from the RFC process.

