---
layout: course
title: Earth Data Science
description: This course covers the foundational aspects of data science, including data collection, cleaning, analysis, and visualization. Students will learn practical skills for working with real-world datasets oriented on earth data (seismic data, Lidar data, satellite images...)
instructor: Antoine Lucas & Alexandre Fournier, Nobuaki Fuji, Grégory Sainton 
year: 2021

term: Fall
location: IPGP, 1 place Jussieu - Paris - Salle Océane - Salle TP 1er étage
course_id: earth-data-science
schedule:
  - week: 1
    topic: Data for Earth science 1/2
    description: Introduction to linear regression, other regression, and ACP.
    materials:
      - name: Lecture Notes
        url: https://pss-gitlab.math.univ-paris-diderot.fr/dralucas/earth-data-science/-/tree/main/Lectures

  - week: 2
    topic: Data for Earth science 2/2
    description: Introduction to inference, inverse problem.
    materials:
      - name: Lecture Notes
        url: https://pss-gitlab.math.univ-paris-diderot.fr/dralucas/earth-data-science/-/tree/main/Lectures

  - week: 3
    topic: Crash course in deep learning (by Grégory Sainton)
    description: Introduction to neural networks, gradient descent, backprop, regularisation...
    materials:
      - name: Lecture Notes
        url: https://github.com/GregS1t/deeplearning101

  - week: 4
    topic: Code Lab 1 - Inspect data and play with regressions
    description: Learn how to clean improper or incomplete data. Apply several regression on them.


  - week: 5
    topic: Code Lab 2 - Inverse problem on the Teil earthquake
    description: Apply inverse problem to find the epicenter of the earthquake

  - week: 6
    topic: Code Lab 3 - LiDAR data analysis and ML segmentation
    description: Apply ML to analyse the multiscale data in LiDAR data

  - week: 7
    topic: Assigment 
    description: Lab in group in 4h


---

## Course Overview

This introductory course on machine learning covers fundamental concepts and algorithms in the field. By the end of this course, students will be able to:

- Understand key machine learning paradigms and concepts
- Implement basic machine learning algorithms using tensorflow and scikit-learn
- Evaluate and compare model performance
- Apply machine learning techniques to earth data

## Prerequisites

- Basic knowledge of linear algebra and calculus
- Programming experience in Python
- Probability and statistics fundamentals

## Textbooks

- Reference: "Machine learning avec Scikit-learn", A. Geron
- Reference: "Pattern Recognition and Machine Learning" by Christopher Bishop
- Reference: "Deep leaning with Tensorflow" by A. Geron
