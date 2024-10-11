==========================
LeRobot X Aloha User Guide
==========================


Setting up Conda Environment
============================

Download and Install Miniconda
------------------------------

To begin, follow the official `Miniconda Installation Guide <https://docs.anaconda.com/miniconda/miniconda-install/>`_ 
to download and install Miniconda on your system.

Environment Setup
-----------------

#. Create a virtual environment:

   .. code-block:: bash

      conda create -n lerobot python=3.10

#. Activate the virtual environment:

   .. code-block:: bash

      conda activate lerobot

Clone Repository
================

For users of Aloha Stationary:

#. Clone the LeRobot repository:

   .. code-block:: bash

      $ cd ~
      $ git clone https://github.com/huggingface/lerobot.git

Build and Install LeRobot Models
================================

#. Build and install the LeRobot models from source:

   .. code-block:: bash

      $ cd lerobot && pip install -e .

#. As we are working with real robots we will require to install dependencies for :guilabel:`intelrealsense` camera's and :guilabel:`dynamixel` servos.

   .. code-block:: bash

      $ cd lerobot && pip install .[intelrealsense,dynamixel]

Teleoperation
=============

To teleoperate your robot, follow these steps:

#. Find the serial numbers of your robot's arms and cameras as described in the following documentation:

   - Arm Symlink Setup: `Aloha Post Install Arm Setup <https://docs.trossenrobotics.com/aloha_docs/getting_started/stationary/software_setup.html#arm-symlink-setup>`_
   - Camera Setup: `Aloha Post Install Camera Setup <https://docs.trossenrobotics.com/aloha_docs/getting_started/stationary/software_setup.html#camera-setup>`_

#. Update the serial numbers in the configuration file: :file:`lerobot/common/configs/robot/aloha.yaml`

#. Run the teleoperation script:

   .. code-block:: bash

      $ python lerobot/scripts/control_robot.py teleoperate \
         --robot-path lerobot/configs/robot/aloha.yaml

   You will see logs that include information such as delta time (dt), frequency, and read/write times for the robot arms.

#. You can control the teleoperation frequency using the :guilabel:`--fps` argument. For example, to set it to 30 FPS:

   .. code-block:: bash

      $ python lerobot/scripts/control_robot.py teleoperate \
         --robot-path lerobot/configs/robot/aloha.yaml --fps 30

Customizing Teleoperation with Hydra
-------------------------------------

You can override the default YAML configurations dynamically using Hydra syntax.
For example, to change the USB ports of the leader and follower arms:

.. code-block:: bash

   $ python lerobot/scripts/control_robot.py teleoperate \
      --robot-path lerobot/configs/robot/aloha.yaml \
      --robot-overrides \
         leader_arms.main.port=/dev/tty.usbmodem575E0031751 \
         follower_arms.main.port=/dev/tty.usbmodem575E0032081


If you don't have any cameras connected, you can exclude them using Hydra's syntax:

.. code-block:: bash

   $ python lerobot/scripts/control_robot.py teleoperate \
      --robot-path lerobot/configs/robot/aloha.yaml \
      --robot-overrides '~cameras'

Recording Data Episodes
=======================

The system supports episode-based data collection, where episodes are time-bounded sequences of robot actions.

#. Control the recording flow with these arguments:

   - :guilabel:`--warmup-time-s`: Number of seconds for device warmup (default: 10s)
   - :guilabel:`--episode-time-s`: Number of seconds per episode (default: 60s)
   - :guilabel:`--reset-time-s`: Time for resetting after each episode (default: 60s)
   - :guilabel:`--num-episodes`: Number of episodes to record (default: 50)

   Example:

   .. code-block:: bash

      $ python lerobot/scripts/control_robot.py record \
         --robot-path lerobot/configs/robot/aloha.yaml \
         --fps 30 \
         --root data \
         --repo-id ${HF_USER}/aloha_test \
         --tags tutorial \
         --warmup-time-s 5 \
         --episode-time-s 30 \
         --reset-time-s 30 \
         --num-episodes 2

.. note::

   #. The :guilabel:`--num-episodes` defines the total number of episodes to be collected.
      Therefore it will check the existing output directories for any previously recorded episodes and will start recording from the last recorded episode.

   #. The recorded data is pushed to hugging face hub by default you can set this false by using :guilabel:`--push-to-hub 0`.

.. note::

   #. To push your dataset to Hugging Face's Hub, log in with a write-access token:

      .. code-block:: bash

         $ huggingface-cli login --token ${HUGGINGFACE_TOKEN} --add-to-git-credential

   #. Set your Hugging Face username as a variable for ease:

      .. code-block:: bash

         $ HF_USER=$(huggingface-cli whoami | head -n 1)

Visualizing Datasets
====================

.. video:: ../videos/visualize_dataset.mp4
   :width: 1280
   :height: 720

To visualize all the episodes recorded in your dataset, run:

.. code-block:: bash

   $ python lerobot/scripts/visualize_dataset_html.py \
      --root data \
      --repo-id ${HF_USER}/aloha_test

To visualize a single dataset episode from the Hugging Face Hub:

.. code-block:: bash

   $ python lerobot/scripts/visualize_dataset.py \
      --repo-id ${HF_USER}/aloha_static_block_pickup \
      --episode-index 0

To visualize a single dataset episode stored locally:

.. code-block:: bash

   $ DATA_DIR='./my_local_data_dir' python lerobot/scripts/visualize_dataset.py \
      --repo-id TrossenRoboticsCommunity/aloha_static_block_pickup \
      --episode-index 0

Replay Recorded Episodes
========================

Replaying episodes allows you to test the repeatability of the robot's actions.
To replay the first episode of your recorded dataset:

.. code-block:: bash

   $ python lerobot/scripts/control_robot.py replay \
      --robot-path lerobot/configs/robot/aloha.yaml \
      --fps 30 \
      --root data \
      --repo-id ${HF_USER}/aloha_test \
      --episode 0

.. tip::

   Use different :guilabel:`--fps` values to adjust the frequency of the robot actions.

Training
========

To train a policy for controlling your robot, use the following command:

.. code-block:: bash

   $ DATA_DIR=data python lerobot/scripts/train.py \
      dataset_repo_id=${HF_USER}/aloha_test \
      policy=act_aloha_real \
      env=aloha_real \
      hydra.run.dir=outputs/train/act_aloha_test \
      hydra.job.name=act_aloha_test \
      device=cuda \
      wandb.enable=false

.. note::

   The arguments are explained below:

   #. We provided the dataset with :guilabel:`dataset_repo_id=${HF_USER}/aloha_test`.
   #. The policy is specified with :guilabel:`policy=act_aloha_real`.
      This configuration is loaded from :file:`lerobot/configs/policy/act_aloha_real.yaml`.
   #. The environment is set with :guilabel:`env=aloha_real`.
      This configuration is loaded from :file:`lerobot/configs/env/aloha_real.yaml`.
   #. The device is set to :guilabel:`cuda` to utilize an NVIDIA GPU for training.
   #. :guilabel:`wandb.enable=true` is used for visualizing training plots via `Weights and Biases <https://docs.wandb.ai/quickstart>`_.
      Ensure you are logged in by running `wandb login`.


Upload Policy Checkpoints
=========================

Once training is complete, upload the latest checkpoint with:

.. code-block:: bash

   $ huggingface-cli upload ${HF_USER}/act_aloha_test \
      outputs/train/act_aloha_test/checkpoints/last/pretrained_model

To upload intermediate checkpoints:

.. code-block:: bash

   $ CKPT=010000
   $ huggingface-cli upload ${HF_USER}/act_aloha_test_${CKPT} \
      outputs/train/act_aloha_test/checkpoints/${CKPT}/pretrained_model

Google Colab for Training
===============================

If you would like to speed up the training process or do not have access to a powerful local machine, you can use the **Google Colab Notebook** that we have prepared for training LeRobot models on a cloud platform.
Colab provides free access to GPUs, which can significantly reduce training time.

To access and use the Colab notebook, follow these steps:

1. **Download or Open the Colab Notebook**: You can either download the Colab notebook to your local machine or open it directly in Google Colab for instant use.

   **Options**:
   
   - :download:`Download the Notebook <../files/LeRobot_Notebook.ipynb>`
   
   - .. raw:: html

      <a target="_blank" href="https://colab.research.google.com/github/TrossenRobotics/aloha_docs/blob/main/docs/files/LeRobot_Notebook.ipynb">
        <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
      </a>

2. **GPU Setup**: Colab allows you to leverage powerful GPUs (e.g., T4, A100) to accelerate the training process.
   Ensure you have enabled GPU by navigating to **Runtime** > **Change runtime type** > **GPU**.

   If you're new to Google Colab or need more information on how it works, check out the `Google Colab FAQ <https://research.google.com/colaboratory/faq.html>`_ for answers to common questions.
   
3. **Install Dependencies**: The notebook will automatically install all necessary dependencies such as `pyrealsense2`, `dynamixel-sdk`, and other tools required for the LeRobot framework.
4. **Log in to Hugging Face**: Follow the instructions to log in with your Hugging Face token for seamless access to datasets and model uploads.
5. **Start Training**: The notebook is pre-configured with commands to start training with the Aloha policy and datasets.
6. **Monitor Progress**: Keep an eye on the first few training epochs to ensure everything runs smoothly.

For additional step-by-step instructions, check out our `instructional video <https://www.youtube.com/watch?v=KAdVobQZSBg>`_.

Benefits of Using Colab
-----------------------

- **GPU Acceleration**: Google Colab provides free access to NVIDIA GPUs, which can dramatically reduce the time needed for model training.
- **Cloud-Based**: You don’t need to rely on your local machine for heavy computation, and the training session can run in the background.
- **Seamless Integration**: The notebook is integrated with Hugging Face, allowing you to easily access datasets and upload trained models.
- **No Setup Hassle**: All the necessary dependencies and configurations are handled within the notebook, making the setup easy and quick.

Evaluation
==========

To control your robot with the trained policy and record evaluation episodes:

.. code-block:: bash

   $ python lerobot/scripts/control_robot.py record \
      --robot-path lerobot/configs/robot/aloha.yaml \
      --fps 30 \
      --root data \
      --repo-id ${HF_USER}/eval_aloha_test \
      --tags tutorial eval \
      --warmup-time-s 5 \
      --episode-time-s 30 \
      --reset-time-s 30 \
      --num-episodes 10 \
      -p outputs/train/act_aloha_test/checkpoints/last/pretrained_model

This command is similar to the one used for recording training datasets, with a couple of key changes:

#. The :guilabel:`-p` argument is now included, which specifies the path to your policy checkpoint (e.g., :guilabel:`-p outputs/train/eval_aloha_test/checkpoints/last/pretrained_model`).
   You can also refer to the model repository on Hugging Face if you have uploaded a model checkpoint there (e.g., :guilabel:`-p ${HF_USER}/act_aloha_test`).

#. The dataset name begins with :guilabel:`eval`, reflecting that you are running inference (e.g., :guilabel:`--repo-id ${HF_USER}/eval_aloha_test`).

You can visualize the evaluation dataset afterward using:

.. code-block:: bash

   $ python lerobot/scripts/visualize_dataset.py \
      --root data \
      --repo-id ${HF_USER}/eval_aloha_test

Trossen Robotics Community
==========================

Pretrained Models
-----------------

You can download pretrained models from the Trossen Robotics Community on Hugging Face and use them for evaluation purposes.
To run evaluation on the pretrained models, use the following command:

.. code-block:: bash

   $ python lerobot/scripts/control_robot.py record \
     --robot-path lerobot/configs/robot/aloha.yaml \
     --fps 30 \
     --root data \
     --repo-id ${HF_USER}/eval_aloha_test \
     --tags tutorial eval \
     --warmup-time-s 5 \
     --episode-time-s 30 \
     --reset-time-s 30 \
     --num-episodes 10 \
     -p ${HF_USER}/act_aloha_test

Datasets for Training and Augmentation
--------------------------------------

Datasets can also be downloaded from the Trossen Robotics Community on Hugging Face for further training or data augmentation.
These datasets can be used with your preferred network architectures.
Instructions for downloading and using these datasets can be found at the following link:

`Dataset Download and Upload Instructions <https://docs.trossenrobotics.com/aloha_docs/operation/hugging_face.html>`_

`Trossen Robotics Community <https://huggingface.co/TrossenRoboticsCommunity>`_

Troubleshooting
===============

.. warning::
   If you encounter issues, follow these troubleshooting steps:


Lag Observed in Follower Arms
-----------------------------

If you notice lag in the follower arms, it's due to the safety settings, which are in place to prevent overshooting that could harm the robot.
These are designed to ensure safety for new users or when using untested policies.

Once you are comfortable with the kit and the trained policy, you can adjust or disable these safety settings by modifying the configuration.

Follow these steps:

#. Open the configuration file located at:

   ``lerobot/configs/robots/aloha.yaml``

#. Locate the following line in the configuration file:

   .. code-block:: yaml

      max_relative_target: 5  # Original value

#. Change the value of `max_relative_target` from `5` to `null` to disable the safety limit:

   .. code-block:: yaml
      :emphasize-lines: 5

      # /!\ FOR SAFETY, READ THIS /!\
      # `max_relative_target` limits the magnitude of the relative positional target vector for safety purposes.
      # The default setting is 5 degrees for Aloha robot motors.
      # Modify this value to null to remove the limit once you feel confident with the robot.
      max_relative_target: null  # Updated value

.. important::

   We recommend starting by teleoperating the grippers (commenting out the rest of the motors in the YAML file).
   Gradually enable additional motors until you can control both arms safely.

OpenCV Installation Issues (Linux)
----------------------------------

   If you encounter OpenCV installation issues, uninstall it via :guilabel:`pip` and reinstall using Conda:

   .. code-block:: bash

      $ pip uninstall opencv-python
      $ conda install -c conda-forge opencv=4.10.0

FFmpeg Encoding Error (:guilabel:`unknown encoder libsvtav1`)
-------------------------------------------------------------

   Install FFmpeg with :guilabel:`libsvtav1` support via Conda-Forge or Homebrew:

   .. code-block:: bash

      $ conda install -c conda-forge ffmpeg

   Or:

   .. code-block:: bash

      $ brew install ffmpeg

Arrow Keys Not Working During Data Recording (Linux)
----------------------------------------------------

   Ensure that the :guilabel:`$DISPLAY` environment variable is set correctly.

Frequency drops during evaluation
---------------------------------

   This happens on low-performance systems due to their inability to handle multi-threaded I/O operations.
   Checkout the following version for a smoother operation.
   Changes will be integrated soon in the newer version of the repository.
   `Low Frequency Fix <https://github.com/Interbotix/lerobot/pull/3>`_

Compute Dataset Statistic Failure
---------------------------------

   It is noticed that on low-performance systems the compute statistic fails due to high batch size and number of workers.
   Checkout the following version with lower batch size and number of workers.
   `Compute Statistic Fix <https://github.com/Interbotix/lerobot/pull/4>`_

Checkout LeRobot Documentation for further help and details
-----------------------------------------------------------

   `LeRobot Github <https://github.com/huggingface/lerobot>`_