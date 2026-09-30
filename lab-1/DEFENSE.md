# Domain Overview: Real-Time Incident & Fight Detection in Video Streams

Automated dangerous incident detection is a specialized branch of computer vision and behavioral video analytics. Its core objective is to continuously monitor live cameras or recorded footage, distinguish normal human interactions from physical fights, and alert monitoring clients within seconds of an occurrence of the incident.

> A single image cannot reliably capture physical violence. Two individuals that suddenly emerge, giving high-five or hugging are often misclassified as act of violance.

Physical violence is fundamentally defined by temporal dynamics of how bodies, limbs, and distances change across frames. Recognizing a fight requires tracking motion, acceleration of body parts, and physical contact patterns over a rolling window of time.

The domain relies on a pipeline that transforms raw sequence of frames to a decision whether the situation is safe or requires an alert.

The pipeline uses RNN (Recurrent Neural Networks) or CNN (Convolutional Neural Network) to analyze frame sequences.
They're trained via existing affray datasets.

The datasets are could be at Kaggle. 