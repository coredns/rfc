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


Why this is a valuable problem to solve? Why is it related to CoreDNS?
Can this be done outside the scope of CoreDNS?

Which place will this proposal live in? A default plugin in [CoreDNS repo](https://github.com/coredns/coredns),
a separate repo in [CoreDNS org](https://github.com/coredns), or a external repo maintained outside of CoreDNS org?
Note exteranl repo maintained outside of CoreDNS org does not need a RFC.

## Design Proposal

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

