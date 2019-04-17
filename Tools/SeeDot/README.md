# SeeDot

SeeDot is an automatic quantization tool that generates efficient machine learning (ML) inference code for IoT devices.

### **Overview**

Most ML models are expressed in floating-point, and IoT devices typically lack hardware support for floating-point arithmetic. Hence, running such ML models on IoT devices involves simulating floating-point arithmetic in software, which is very inefficient. SeeDot addresses this issue by generating fixed-point code with only integer operations. To this end, SeeDot takes as input trained floating-point models (like Bonsai or ProtoNN) and generates efficient fixed-point code that can run on microcontrollers. The SeeDot compiler uses novel compilation techniques like automatically inferring certain parameters used in the fixed-point code, optimized exponentiation computation, etc. With these techniques, the generated fixed-point code has comparable classification accuracy and performs significantly faster than the floating-point code.

To know more about SeeDot, please refer to our paper [here](https://www.microsoft.com/en-us/research/publication/compiling-kb-sized-machine-learning-models-to-constrained-hardware/).

This document describes the tool usage with an example.

### **Software requirements**

1. [**Python 3**](https://www.python.org/) with following packages:
   - **[Antrl4](http://www.antlr.org/)** (antlr4-python3-runtime; tested with version 4.7.2)
   - **[Numpy](http://www.numpy.org/)** (tested with version 1.16.2)
   - **[Scikit-learn](https://scikit-learn.org/)** (tested with version 0.20.3)
2. Linux packages:
   - gcc (tested with version 7.3.0)
   - make (tested with version 4.1)

### **Usage**

SeeDot can be invoked using **`SeeDot.py`** file. The arguments for the script are supplied as follows:

```
usage: SeeDot.py [-h] [-a] --train  --test  --model  [--tempdir] [-o]

optional arguments:
  -h, --help      show this help message and exit
  -a , --algo     Algorithm to run ('bonsai' or 'protonn')
  --train         Training set file
  --test          Testing set file
  --model         Directory containing trained model (output from
                  Bonsai/ProtoNN trainer)
  --tempdir       Scratch directory for intermediate files
  -o , --outdir   Directory to output the generated Arduino sketch
```

An example invocation is as follows:
```
python SeeDot.py -a bonsai --train path/to/train.npy --test path/to/test.npy --model path/to/Bonsai/model
```

SeeDot expects `train` and `test` data files in a specific format. The shape of each data file should be `[numberOfDataPoints, numberOfFeatures + 1]`, where the class label is in the first column. We currently support multiple formats for the data files: numpy arrays (.npy), tab-separated values (.tsv), comma-separated values (.csv), libsvm (.txt).

The `model` directory contains the output of Bonsai/ProtoNN trainer. After training, the learned parameters are stored in a output directory in a specific format. For Bonsai, the learned parameters are `Z`, `W`, `V`, `T`, `Sigma`, `Mean`, and `Std`. For ProtoNN, the learned parameters are `W`, `B`, and `Z`. The parameters can be either numpy arrays (.npy) or in plaintext.

The `tempdir` directory is used to store the intermediate files generated by the compiler. The device-specific fixed-point code is stored in the `outdir` directory.

### Getting started: Quantizing ProtoNN on usps10

To help get started with SeeDot, please follow the below instructions to generate fixed-point code for ProtoNN algorithm on the usps10 dataset. This process consists of three steps: training ProtoNN on usps10, quantizing the trained model with SeeDot, and performing prediction on the device.

#### **Training ProtoNN on usps10** 

1. Clone the EdgeML repository and navigate to the right directory.
     ```
     git clone https://github.com/Microsoft/EdgeML
     cd EdgeML/tf/examples/ProtoNN
     ```

2. Fetch usps10 data.
     ```
     python fetch_usps.py
     python process_usps.py
      ```

3. Invoke ProtoNN trainer using the following command.
      ```
      python protoNN_example.py --data-dir./usps10 --projection-dim 25 --num-prototypes 60 --epochs 100 -o output
      ```
  This should give around 91.12% classification accuracy. The trained model is stored in the `output` directory.

More information on using the ProtoNN trainer can be found [here](https://github.com/Microsoft/EdgeML/tree/master/tf/examples/ProtoNN).

#### **Quantizing with SeeDot**

1. Navigate to the right directory
      ```
      cd ../../../Tools/SeeDot
      ```

2. Invoke SeeDot using the following command.
      ```
      python SeeDot.py -a protonn --train ../../tf/examples/ProtoNN/usps10/train.npy --test ../../tf/examples/ProtoNN/usps10/test.npy --model ../../tf/examples/ProtoNN/usps10/output -o arduino
      ```

   The SeeDot-generated code should give around 91.23% classification accuracy. The difference in classification accuracy is 0.11% compared to the floating-point code. The generated code is stored in the `arduino` folder which contains the sketch along with two files: model.h and predict.cpp. `model.h` contains the quantized model and `predict.cpp` contains the inference code.

#### **Prediction on the device**

Follow the below steps to perform prediction on the device where the SeeDot-generated code is run on a single data-point which is stored on the devices's flash memory.

1. Open the sketch in the [Arduino IDE](https://www.arduino.cc/en/main/software).
2. Connect the Arduino device to the computer and choose the correct board configuration.
3. Upload the sketch to the device.
4. Open the Serial Monitor and select baud rate specified in the sketch (default is 115200) to monitor the output.
5. The average prediction time is computed every 100 iterations.



The above workflow has been tested on Arduino Uno and Arduino MKR1000. It is expected to work on other Arduino devices as well.