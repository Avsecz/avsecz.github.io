---
layout: post
title: Setting up TensorFlow 0.9 with Python 3.5 on AWS GPU-instance
excerpt_separator: <!--more-->
---

This guide will show you how to:

- setup an AWS account on your linux machine
- launch a gpu-powered ec2 instance (`g2.2xlarge`) from the command line
- ssh to the launched instance
- set up tensorFlow 0.9

## Things to be installed

Locally:

- AWS command line tool: `awscli`


On the ec2 instance running Ubuntu 14.04:

- Required linux packages
- CUDA 7.5
- cuDNN v4
- Anaconda with Python 3.5
- TensorFlow 0.9
- GPU usage tool `gpustat`

Tutorial ends by runnning an example MNIST image classifier on a GPU.

<!--more-->

## Inspiration for this guide

I used code and ideas mainly from the following two blog-posts:

- <http://ramhiser.com/2016/01/05/installing-tensorflow-on-an-aws-ec2-instance-with-gpu-support/>
- <http://eatcodeplay.com/installing-gpu-enabled-tensorflow-with-python-3-4-in-ec2/>

<!-- Difference to these guides is that I haven't installed TensosrFlow from source. -->
<!-- - https://github.com/BVLC/caffe/wiki/Install-Caffe-on-EC2-from-scratch-(Ubuntu,-CUDA-7,-cuDNN-3) -->

## Setting up AWS on your local machine

### Register an account at the AWS

First, register an account on the Amazon web services page: <https://aws.amazon.com/>.

### Install AWS command line tool

<!-- I helped myself with two good guides:  -->
<!-- - http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html -->

Next, install the aws command-line tool through python's pip installer ([more info](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)):

```bash
sudo pip install awscli
```


This will provide you with the `aws` command:

```bash
$ aws
usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]
To see help text, you can run:

  aws help
  aws <command> help
  aws <command> <subcommand> help
aws: error: too few arguments
```

You can enable the auto-complete function in bash/zsh by following [this](https://github.com/aws/aws-cli#command-completion) guide.

### Configure AWS

In order to administrate our aws account, we have to provide the right credentials:

1. Create a new aws user [here](https://console.aws.amazon.com/iam/home?#users) and download the credentials .csv file

		User Name,Access Key Id,Secret Access Key
		"some_user",<acces_key_id>,<secret_access_key>

2. Run `aws configure` and provide the credentials obtained in the previous step:


		$ aws configure
		AWS Access Key ID: <acces_key_id>
		AWS Secret Access Key: <secret_access_key>
		Default region name [us-east-1]: us-east-1
		Default output format [None]: <ENTER>


	Configuration will get stored in the directory `~/.aws`.

3. Give the created user admin rights: follow [this](http://stackoverflow.com/questions/28222445/aws-cli-client-unauthorizedoperation-even-when-keys-are-set/31323864#31323864) stackoverflow answer


4. Test your installation and configuration by running:

    ```bash
    $ aws ec2 describe-instances --output table
    -------------------
    |DescribeInstances|
    +-----------------+
    ```

	I had to wait a minute or so after enabling the admin rights. Before that, I was getting the 'unauthorized' error:


		Client.UnauthorizedOperation: You are not authorized to perform this operation. (Service: AmazonEC2; Status Code: 403; Error Code: UnauthorizedOperation; ...

4. Create an access group `my-sg` and set access rightswith ssh access rights

		# create my-sg group
		aws ec2 create-security-group --group-name my-sg \
			--description "My security group"
	
        # enable ssh access on port 22 from any IP address
		aws ec2 authorize-security-group-ingress --group-name my-sg \
			--protocol tcp --port 22 --cidr 0.0.0.0/0

	Note that this access group doesn't impose any IP filter for the ssh access: `--cidr 0.0.0.0/0`.


5. Create the ssh access key and save it to `~/.aws/my_aws_key.pem`


		aws ec2 create-key-pair --key-name my_aws_key --query 'KeyMaterial' --output text > ~/.aws/my_aws_key.pem
		chmod 400 ~/.aws/my_aws_key.pem

Alright! We are now ready to launch the ec2 instance!

## Launch Ubuntu 14.04 GPU instance

We will launch Ubuntu 14.04 instance (image-id = ami-fce3c696, for more basic images see the [AWS LaunchInstanceWizard](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#LaunchInstanceWizard). 

```bash
aws ec2 run-instances --image-id ami-fce3c696 \
	--count 1 --instance-type g2.2xlarge \
	--key-name my_aws_key \
	--security-groups my-sg
```

## SSH to the instance

In order to get the IP of all the running instances, we can create ourselves an `aws_get_ip` alias and put it into `~/.bashrc`:

```bash
alias aws_get_ip='aws ec2 describe-instances --query "Reservations[*].Instances[*].PublicIpAddress" --output=text'
```

```bash
$ aws_get_ip
<instance IP>
```

Finally, we can connect to the instance through SSH:

```bash
$ aws_get_ip
<instance IP>
ssh -i ~/.aws/my_aws_key.pem ubuntu@<instance IP>
```
Since we used the ubuntu AMI image, the default user is `ubuntu`. For Amazon's AMI, the user would is `ec2-user`.

## Install TensorFlow requirements

### Install CUDA 7.5

```bash
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get -y install linux-headers-$(uname -r) linux-image-extra-`uname -r`
wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1404/x86_64/cuda-repo-ubuntu1404_7.5-18_amd64.deb
sudo dpkg -i cuda-repo-ubuntu1404_7.5-18_amd64.deb
rm cuda-repo-ubuntu1404_7.5-18_amd64.deb
sudo apt-get update
sudo apt-get install -y cuda
```

You should now reboot your machine, even though it worked for me without rebooting:

```bash
sudo reboot
```

### Verify CUDA installation

```bash
$ nvidia-smi
Fri Jun 17 21:25:59 2016
+------------------------------------------------------+
| NVIDIA-SMI 352.93     Driver Version: 352.93         |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|============================+======================+===================|
|   0  GRID K520           Off  | 0000:00:03.0     Off |                  N/A |
| N/A   26C    P0    36W / 125W |     11MiB /  4095MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

### Install cuDNN v4

Register and download the cuDNN v4 from [here](https://developer.nvidia.com/rdp/cudnn-download). You can then put it into your dropbox folder and share the link:

```bash
CUDNN_FILE=cudnn-7.0-linux-x64-v4.0-prod.tgz
wget https://www.dropbox.com/s/.../${CUDNN_FILE}
tar xvzf ${CUDNN_FILE}
rm ${CUDNN_FILE}
sudo cp cuda/include/cudnn.h /usr/local/cuda/include # move library files to /usr/local/cuda
sudo cp cuda/lib64/libcudnn* /usr/local/cuda/lib64
sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*
rm -rf ~/cuda
```

### Setup the apropriate library paths

```bash
echo 'export CUDA_HOME=/usr/local/cuda
export CUDA_ROOT=/usr/local/cuda
export PATH=$PATH:$CUDA_ROOT/bin:$HOME/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$CUDA_ROOT/lib64
' >> ~/.bashrc
```

## Install Python and Tensorflow

### Install Anaconda with python 3.5

```bash
wget http://repo.continuum.io/archive/Anaconda3-4.0.0-Linux-x86_64.sh
bash Anaconda3-4.0.0-Linux-x86_64.sh -b -p ~/bin/anaconda3
rm Anaconda3-4.0.0-Linux-x86_64.sh
echo 'export PATH="$HOME/bin/anaconda3/bin:$PATH"' >> ~/.bashrc
```

### Setup tensorFlow

```bash
export TF_BINARY_URL=https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow-0.9.0rc0-cp35-cp35m-linux_x86_64.whl
sudo pip install $TF_BINARY_URL

# I got an error with the `--upgrade` flag:
#  sudo pip install --upgrade $TF_BINARY_URL
```

I used install without the `--upgrade` flag as it gave me an error: 

```
Cannot remove entries from nonexistent file /home/ubuntu/bin/anaconda2/lib/python2.7/site-packages/easy-install.pth
```


## Run a MNIST classifier and monitor the system usage

To finish the installation process, let's run a MNIST classifier and monitor the system usage.

First, install the required packages:

```bash
sudo apt-get install htop
sudo wget https://git.io/gpustat -O /usr/local/bin/gpustat
sudo chmod +x /usr/local/bin/gpustat
sudo nvidia-smi daemon  ## run daemon to make monitoring faster
```

Now start [byobu](http://byobu.co/) (terminal multiplexer, similar to [tmux](https://tmux.github.io/) or [GNU screen](https://www.gnu.org/software/screen/)):

```bash
byobu
```

Next, press `Ctrl-F2` to split the window vertically and run htop:

```bash
htop
```

Press `Shift-F2` to split the window horizontally and run the continous GPU monitor `gpustat`:

```bash
watch --color -n1.0 gpustat -cp
```

With `Shift-<left>` move to the left pannel, download the MNIST classification script and execute it:

```bash
wget https://raw.githubusercontent.com/tensorflow/tensorflow/master/tensorflow/models/image/mnist/convolutional.py
python convolutional.py
```

Congrats! You've made it!

![Alt text](/images/aws-tensorflow.png "Running the mnist model on aws.")

