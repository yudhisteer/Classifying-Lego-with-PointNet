# Classifying Lego with PointNet

## Problem Statement

## Dataset

## Abstract

## Plan of Action
1. [Understanding PointNet](#up)
2. [Coding PointNet](#cp)
3. [Training with ShapeNet Dataset](#tsd)
4. [Training with Custom Dataset](#tcd)

-----------------
<a name="up"></a>
## 1. Understanding PointNet

Before PointNet, researchers would convert point cloud data into **3D Voxel Grids**. A large portion of these voxel grids can be empty and this makes the data voluminous. 

PointNet is a neural network architecture designed for processing and understanding point cloud data. The idea behind PointNet is to take in directly the point cloud data such that it respects the ```permutation invariance``` of points in the point cloud data.

<p align="center">
  <img src="https://github.com/yudhisteer/Deep-Point-Clouds-3D-Perception/assets/59663734/f54fd8a0-901c-4743-ac91-a47303888b70" width="70%" />
</p>

The architecture is designed to directly process **unordered** point cloud data, which makes it a useful tool for various ```3D tasks``` such as ```object classification```, ```object segmentation```, and ```shape recognition```. 

<p align="center">
  <img src="https://github.com/yudhisteer/Deep-Point-Clouds-3D-Perception/assets/59663734/3d3487cf-65c5-4cc7-b744-5813d5b6e27d" width="40%" />
</p>


### 1.1 Permutation Invariance

Point clouds represent a set of points in 3D space. Each point typically has attributes such as position ```(x, y, z)``` and possibly additional features (e.g., ```color, intensity```). These point clouds are **unordered** sets of points, meaning that the order of the points in the set doesn't convey any meaningful information. For example, if we have a point cloud representing the 3D coordinates of objects in a scene, we could shuffle the order of the points without changing the scene itself. Here's an example below:

**Original Point Cloud:**
 
```python
Point 1: (0, 0, 0)
Point 2: (1, 0, 0)
Point 3: (0, 1, 0)
Point 4: (1, 1, 0)
Point 5: (0, 0, 1)
Point 6: (1, 0, 1)
Point 7: (0, 1, 1)
Point 8: (1, 1, 1)
```

**Shuffled Point Cloud:**

```python
Point 1: (0, 1, 0) # Previously point 3
Point 2: (1, 0, 0) # Previously point 2
Point 3: (1, 1, 0) # Previously point 4
Point 4: (0, 0, 0) # Previously point 1
Point 5: (0, 1, 1) # Previously point 1
Point 6: (1, 1, 1) # Previously point 8
Point 7: (1, 0, 1) # Previously point 6
Point 8: (0, 0, 1) # Previously point 5
```

Although the order of the points has changed, the ```spatial relationships``` between the points and the overall structure of the cube **remain the same**. It's about recognizing that the order of points in a point cloud **doesn't change** the ```underlying geometry``` or content being represented.


#### How does PointNet process point cloud data in a permutation-invariant manner?

- **Symmetric Functions:** ```Max-pooling``` to aggregate information from all points in the input set. The operation treats all points equally, regardless of their order.

- **Shared Weights:** The same set of weights using ```Multi-Layer Perceptron (MLP)``` is used for all points in the input. This means that the processing applied to one point is the same as that applied to any other point. This shared weight scheme ensures that the network doesn't favor any particular order of points.

### 1.2 Point Cloud Properties
Let's describe the three main properties of a point cloud:

- **Unordered:** A point cloud is a collection of 3D points, and unlike images or grids, there's ```no specific order``` to these points. This means that the order in which we feed the points to a network shouldn't matter. It should be able to handle any order we provide.

- **Interaction Among Points:** These points aren't isolated; they have a ```distance metric```. Nearby points often form meaningful structures. So, a good model should be able to capture these local patterns and understand how points interact with each other.

- **Invariance Under Transformations:** A good representation of a point cloud should stay the same even if we ```rotate``` or ```translate``` the entire point cloud. In other words, changing the viewpoint or position of the points as a whole shouldn't change the global point cloud category or segmentation of the points.

### 1.3 Input Transform
From the PointNet architecture, we observe that the Input Transform encompasses a ```T-Net```. So what is a T-Net? The T-Net is a type of **Spatial transformer Network (STN)** that can be seen as a ```mini-PointNet```. The first T-Net takes in ```raw point cloud data``` and outputs a ```3 x 3``` matrix.

<p align="center">
  <img src="https://github.com/yudhisteer/Classifying-Lego-with-PointNet/assets/59663734/bd6df400-744f-468e-ad68-5071db6675f6" width="100%" />
</p>

The T-Net is responsible for predicting an ```affine transformation matrix``` that aligns the input point cloud to a ```canonical space```. This alignment ensures that the network is **invariant** to certain geometric transformations, such as ```rigid transformations```.

Let's take a step back and define some of these complex technical terms:

- **Affine transformation matrix:** This matrix defines how the input points should be ```transformed``` to align with the reference space (Canonical space).


- **Canonical space:** It refers to a ```standardized``` and ```consistent reference point``` for processing 3D point clouds. It's like choosing a common starting point or orientation for all input point clouds. This standardization ensures that no matter how a point cloud is initially positioned or rotated, it gets transformed into this ```common reference frame```. This simplifies the learning process of the neural network. 


- **Rigid transformations:**  Transformation that **preserves** the ```shape``` and ```size``` of an object. It includes operations like ```translation``` (moving an object without changing its shape), ```rotation``` (turning an object around a point), and ```reflection``` (flipping an object). Essentially, rigid transformations **don't distort** or **deform** the object; they only change its ```position``` and ```orientation``` in space.

In summary, this means that no matter how the original point cloud was ```oriented``` or ```positioned```, the T-Net will transform it in such a way that it becomes ```standardized```, making it **easier** for subsequent layers of the network to process the data effectively.

### 1.4 Feature Transform
The PointNet has a second transformer network called ```Feature T-Net```.  The role of this second T-Net is to predict a ```feature transformation matrix``` to align features from different input point clouds. It captures ```fine-grained``` information about ```point-specific transformations``` that are important for capturing ```local details``` and ```patterns```.

The second transformer network has the same architecture as the first except that this one's output is a ```64 x 64``` matrix. In the diagram below we used ```k``` where k will be equal to ```64```.

<p align="center">
  <img src="https://github.com/yudhisteer/Deep-Point-Clouds-3D-Perception/assets/59663734/0f3ebf0d-3eb2-4a1c-a68a-dfcce2b3c31a" width="100%" />
</p>


In the paper, the author argues that the feature space has a ```higher dimension``` which greatly increases the complexity of the optimization process. Hence, they add a ```regularization``` term to their ```softmax``` training loss in order to constrain the feature transformation matrix to be close to ```orthogonal```. By being orthogonal. it helps ensure that the transformation applied to features during alignment doesn't introduce ```unnecessary distortion``` which could result in ```loss of information```. Below is the regularization term where ```A``` is the feature transformation matrix and ```I``` is the identity matrix: 

<p align="center">
  <img src="https://github.com/yudhisteer/Classifying-Lego-with-PointNet/assets/59663734/af430dae-0c60-4929-8b4a-b2e727dae109" />
</p>


In summary, this is the difference between the Input T-Net and the Feature T-Net:

- **Input T-Net:** By aligning the entire input point cloud to a **canonical space**, the Input T-Net captures ```global features``` that are **invariant** to transformations like **rotation** and **translation**.

- **Feature T-Net:** Operates on the **feature vectors** extracted from the point cloud **after** the initial global alignment by the **Input T-Net**. It aims to capture ```local features``` and ```fine-grained patterns``` within the aligned point cloud.

### 1.5 Shared Multi-Layer Perceptron (MLP)
Since the Input T-Net and Feature T-Net are themselves mini-PointNet, they both contain shared MLPs in their architecture. While MLP is a type of neural network architecture that consists of multiple layers of artificial neurons (perceptrons), we will use ```1D convolutional layers (Conv1D)``` for the shared MLP followed by ```Fully-Connected (FC)``` layers in both the Input and Feature Transformation networks. However, we will use only Fully Connected layers for the PointNet after the Feature Transformation network.

Note that the shared MLPs are the ```core feature extraction``` component after the initial alignment and transformation by the Input T-Net and Feature T-Net.

<p align="center">
  <img src="https://github.com/yudhisteer/Classifying-Lego-with-PointNet/assets/59663734/f81e2386-7e43-45a5-b70f-b6d1ab6ae962" width="100%" />
</p>

In the full architecture of the PointNet, we can clearly see how the Input Transformation Network and the Feature Transformation Network are ```mini-PointNets``` themselves.


------------------------
<a name="cp"></a>
## 2. Coding PointNet
Now that we know how PointNet works, we need to code it. Since it is a big neural network architecture, we will divide it into four segments:

1. Input T-Net
2. Feature T-Net
3. PointNet Feat
4. Classification Head

Note that we are only concerned with **classification** for this project and not **segmentation**. If the latter was considered, we would have a fifth segment called Segmentation Head. 

### 2.1 Input T-Net
As explained above, the Input Transform section of PointNet contains a T-Net. We will write a function for the T-Net. Below is the schema for the architecture consisting of 1D Convolutional Layers and Fully Connected layers as shared MLPs. 

```python
id1[Input Point Cloud] --> id2[Conv1D 64 + BatchNorm + ReLU]
id2 --> id3[Conv1D 128 + BatchNorm + ReLU]
id3 --> id4[Conv1D 1024 + BatchNorm + ReLU] 
id4 --> id5[MaxPool]
id5 --> id6[Flatten 1024]
id6 --> id7[FC 512 + BatchNorm + ReLU]
id7 --> id8[FC 256 + BatchNorm + ReLU]
id8 --> id9[FC 9]
id9 --> id10[Add Identity 9]
id10 --> id11[Reshape 3x3]
id11 --> id12[Output Transform Matrix]
```


We first define the **convolutional layers**, **fully connected layers**, **activation function**, and **batch normalization layers** needed:

```python
# Conv layers
conv1 = nn.Conv1d(3, 64, 1)
conv2 = nn.Conv1d(64, 128, 1)
conv3 = nn.Conv1d(128, 1024, 1)
```

```python
# Fully connected layers
fc1 = nn.Linear(1024, 512)
fc2 = nn.Linear(512, 256)
fc3 = nn.Linear(256, 9)
```

```python
# Nonlinearities
relu = nn.ReLU()
```

```python
# Batch normalization layers
bn1 = nn.BatchNorm1d(64)
bn2 = nn.BatchNorm1d(128)
bn3 = nn.BatchNorm1d(1024)
bn4 = nn.BatchNorm1d(512)
bn5 = nn.BatchNorm1d(256)
```

The **function** for T-Net:

```python
def input_tnet(x):
    # Input shape
    batchsize = x.size()[0]
    print("Shape initial:", x.shape)

    # Conv layers
    x = F.relu(bn1(conv1(x)))
    print("Shape after Conv1:", x.shape)

    x = F.relu(bn2(conv2(x)))
    print("Shape after Conv2:", x.shape)

    x = F.relu(bn3(conv3(x)))
    print("Shape after Conv3:", x.shape)

    # Pooling layer
    x = torch.max(x, 2, keepdim=True)[0]
    print("Shape after Max Pooling:", x.shape)

    # Reshape for FC layers
    x = x.view(-1, 1024)
    print("Shape after Reshape:", x.shape)

    # FC layers
    x = F.relu(bn4(fc1(x)))
    print("Shape after FC1:", x.shape)

    x = F.relu(bn5(fc2(x)))
    print("Shape after FC2:", x.shape)

    x = fc3(x)
    print("Shape after FC3:", x.shape)

    # Initialize identity transformation matrix
    iden = Variable(torch.from_numpy(np.array([1, 0, 0, 0, 1, 0, 0, 0, 1]).astype(np.float32))).view(1, 9).repeat(
        batchsize, 1)

    # Move to GPU if needed
    if x.is_cuda:
        iden = iden.cuda()

    # Add identity matrix to transform pred
    x = x + iden

    # Reshape to 3x3
    x = x.view(-1, 3, 3)
    print("Shape after Reshape to 3x3:", x.shape)

    return x

```

We will create simulated data for the point cloud with the following parameters:

- batch size = ```32```
- number of points = ```2500```
- number of channels (x,y,z) = ```3```

```python
    # Generate a sample input tensor (N, 3, 2500)
    sample_input = torch.randn(32, 3, 2500)
```


Below is the size of the layers after each process. The shape of the layer after T-Net and before matrix multiplication is a ```3 x 3``` matrix.


```python
Shape initial: torch.Size([32, 3, 2500])
Shape after Conv1: torch.Size([32, 64, 2500])
Shape after Conv2: torch.Size([32, 128, 2500])
Shape after Conv3: torch.Size([32, 1024, 2500])
Shape after Max Pooling: torch.Size([32, 1024, 1])
Shape after Reshape: torch.Size([32, 1024])
Shape after FC1: torch.Size([32, 512])
Shape after FC2: torch.Size([32, 256])
Shape after FC3: torch.Size([32, 9])
Shape after Reshape to 3x3: torch.Size([32, 3, 3])
```

### 2.2 Feature T-Net


```python
id1[Input Point Cloud] --> id2[Conv1D 64 + BatchNorm + ReLU]
id3 --> id4[Conv1D 128 + BatchNorm + ReLU] 
id5 --> id6[Conv1D 1024 + BatchNorm + ReLU]
id7 --> id8[MaxPool]
id9 --> id10[Flatten] 
id11 --> id12[FC 512 + BatchNorm + ReLU]
id13 --> id14[FC 256 + BatchNorm + ReLU]  
id15 --> id16[FC k*k]
id17 --> id18[Add Identity]
id18 --> id19[Reshape k x k]
id19 --> id20[Output Transform]
```

### 2.3 PointNet Feature





### 2.4 Classification Head


### 2.5 Segmentation Head








------------------------
<a name="tsd"></a>
## 3. Training with ShapeNet Dataset

------------------------
<a name="tcd"></a>
## 4. Training with Custom Dataset

-------------------

## References
1. https://medium.com/@luis_gonzales/an-in-depth-look-at-pointnet-111d7efdaa1a
2. https://github.com/luis-gonzales/pointnet_own
3. https://keras.io/examples/vision/pointnet/
4. https://medium.com/p/90398f880c9f
5. https://towardsdatascience.com/3d-deep-learning-python-tutorial-pointnet-data-preparation-90398f880c9f
6. https://medium.com/deem-blogs/speaking-code-pointnet-65b0a8ddb63f
7. https://arxiv.org/pdf/1506.02025.pdf
8. https://medium.com/geekculture/understanding-3d-deep-learning-with-pointnet-fe5e95db4d2d
9. https://towardsdatascience.com/deep-learning-on-point-clouds-implementing-pointnet-in-google-colab-1fd65cd3a263
10. https://github.com/nikitakaraevv/pointnet/blob/master/nbs/PointNetClass.ipynb
11. https://medium.com/@itberrios6/introduction-to-point-net-d23f43aa87d2
12. https://github.com/fxia22/pointnet.pytorch/blob/master/pointnet/model.py
13. https://medium.com/@itberrios6/point-net-from-scratch-78935690e496
14. https://medium.com/@luis_gonzales/an-in-depth-look-at-pointnet-111d7efdaa1a
15. https://www.youtube.com/watch?v=HIRj5pH2t-Y&t=427s&ab_channel=Lights%2CCamera%2CVision%21
16. https://towardsdatascience.com/a-comprehensive-introduction-to-different-types-of-convolutions-in-deep-learning-669281e58215
