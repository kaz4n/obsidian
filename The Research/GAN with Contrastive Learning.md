What we have:
- Lightweight 
- contrastive learning ensures the robustness and reliability of the IDS and circumvents the need for large training data or for tuning the hyperparameters for different datasets.
- use of an autoencoder-based architecture, which is a self-supervised learning technique that overcomes the challenge of requiring labeled data.
- However, in unsupervised intrusion detection, the goal is to learn an appropriate representation of the normal data and  detect anomalies rather than generating realistic instances. Hence, EGBAD relies on an autoencoder-based architecture that maps the input samples to a representative latent space without using a discriminator
- in EGBAD, the decoder part of the autoencoder is omitted, as the compressed form maps the data into the  embedding space that is needed for detection. This simplification reduces the complexity of the GAN and makes it more applicable to resource-constrained devices.
- After the model is trained on normal traffic data, its components are adapted for deployment in IoT environments. During training, the generator implicitly contributes to constructing the latent representation of normal data, while the discriminator is omitted to minimize computational cost. This adaptation allows the model to learn the intrinsic structure of benign data efficiently without the need for adversarial competition between components. In the inference phase, only the encoder is retained as the primary component and deployed on IoT devices, where it efficiently maps new inputs into the latent space for anomaly detection.
- In the context of this work, where the model training considers a subset of normal traffic data, the proposed contrastive learning approach considers only positive pairs. Hence, the contrastive loss minimizes the distance  between training data points by encouraging the model to learn a representation where normal data points are close to each other in the embedding space. This process enables the model to capture the underlying distribution of benign network behavior, allowing it to recognize deviations as potential anomalies even without explicit negative samples.

We want: 
- keep it unsupervised
- No retraining for each device class and adaptability to unseen anomalies due to their supervised nature, to identify zero-day attacks or new attack patterns.
- We want a system which clusters abnormal data into cluster, the system should adapt to different scenarios and systems, for example in some systems there's only 5 abnormal types, in another scenario there might be 9 abnormal types, we want the system to determine it in different scenario.

Research the past used models, and find a gap or unresearched part which we could leverage to to begin our research, 


