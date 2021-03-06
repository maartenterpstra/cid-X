Usage
=====



Quick start
-----------
Fill in the configuration file ``XCAT_dvf_processing.cfg`` (see here: :ref:`config-file`) and call the main script with the config file as a parameter.

.. code-block:: none

   python XCATdvfProcessing.py XCAT_dvf_processing.cfg

A detailed description of how to configure the processing is given below



Processing steps
----------------

The processing pipeline can be divided into three separate parts, namely 

* **pre-processing** (see: :ref:`pre-processing`), 
* **batch DVF post-processing** (see: :ref:`batch-dvf-processing`), and 
* **batch image warping** (see: :ref:`batch-warping`). 

By setting the corresponding parameters in the first section of the configuration file, the different steps are enabled or disabled. It is highly recommended to run each individual step and check the results before continuing with the next. 

.. code-block:: python
   
   [PROCESSING_STEPS]
   doPreProcessing       = 1
   doBatchPostProcessing = 0
   doBatchWarping        = 0 

.. _pre-processing:

Pre-processing
^^^^^^^^^^^^^^

The most important output of the pre-processing is the *signed distance map* that defines the sliding regions in the first time point. To calculate this, first a rough segmentation using the information provided by the XCAT DVF-text file is generated. This is an incomplete segmentation (since organs are defined by surfaces), the sliding region is thereafter calculated by a level-set evolution. 

.. note:: 
   
   Note that the level-set evolution can take some time to complete.


.. code-block:: python

   [PREPROCESSING]
   xcatAtnFile       = <Complete/Path/to/the/XCAT/attenuation/file.bin>
   xcatDVFFile       = <Complete/Path/to/the/XCAT/DVF/file.txt>
   outDir            = <Path/to/where/the/results/of/the/pre-processing/step/will/be/written/>
   outDistMapImgName = <fileNameOfTheSignedDistanceMap.nii.gz>      # will be written to [PREPROCESSING][outDir]
   outXCATAtnImgName = <fileNameOfTheXCATAttenuationFile.nii.gz>    # will be written to [PREPROCESSING][outDir]
   numVoxX           = <imageDimensionInX>
   numVoxY           = <imageDimensionInY>
   numVoxZ           = <imageDimensionInZ>
   spacingX          = <voxelSpacingInX>
   spacingY          = <voxelSpacingInY>
   spacingZ          = <voxelSpacingInZ>
   
   # Optional, comment if not using
   outXCATCTImgName           = <fileNameOfTheXCATImageInHU.nii.gz>   # will be written to [PREPROCESSING][outDir]
   numberOfLevelSetIterations = <numberOfLevelSetIterations>          # Values depend on the field of view, the image resolution etc. 
                                                                      # we so far had good experiences with values between 5000 and 7500
   outLevelSetImgName         = <fileNameOfTheLevelSetImage.nii.gz>   # will be written to [PREPROCESSING][outDir]
   saveLevelSetImage          = 1                                     # Set to 1 if you would like the level-set image to be saved, to 0 otherwise


.. _batch-dvf-processing:

Batch DVF post-processing
^^^^^^^^^^^^^^^^^^^^^^^^^^

The batch processing uses the signed distance map from the previous step to calculate the sliding-preserving inversion of the XCAT DVF. The output of this step are the forward and backward DVFs that can are used as the new ground-truth deformations. 

One final pre-processing step is carried out for the first iteration, which calculates the spatial gradient of the signed distance map at the first simulated time point. Only the output file names need to be specified and if these files do not exist in the pre-processing output directory these will be automatically generated.


.. code-block:: python

   [BATCH_POSTPROCESSING]
   xcatDVFFilePattern   = <Complete/Path/and/pattern/of/all/XCAT/DVF_files_vec_frame1*.txt>   # Use the wildcard * to enable listing all DVF files
   outDir               = <Path/to/where/the/results/of/the/batch-dvf-processing/step/will/be/written/>
   tmpDir               = <Path/to/where/temporary/filest/of/the/batch-dvf-processing/step/will/be/written/>
   outDistMapDxImgName  = <fileNameOfTheSDT_gradientInX.nii.gz>    # maybe something like this: distMap_dx.nii.gz
   outDistMapDyImgName  = <fileNameOfTheSDT_gradientInY.nii.gz>    # maybe something like this: distMap_dy.nii.gz
   outDistMapDzImgName  = <fileNameOfTheSDT_gradientInZ.nii.gz>    # maybe something like this: distMap_dz.nii.gz
   corruptedFilesDir    = <Path/to/where/potentially/corrupted/XCAT/text/files/will/be/written/>
   numProcessorsDVFInv  = <NumberOfProcessorsUsedByNiftyReg>       # Integer value
   
   # Optional, comment if not using
   niftyRegBinDir       =  <Path/to/where/the/niftyReg/binaries/are/installed/>

XCAT DVF text files that cannot be completely processed will be moved to the path defined in ``[BATCH_POSTPROCESSING][corruptedFilesDir]``. Incomplete processing can be caused by corrupted XCAT-text files or by other errors occuring during the processing.


.. _batch-warping:

Batch warping
^^^^^^^^^^^^^

In this step the deformation vector fields produced in the previous step are used to transform/warp the first time point to all other time points.

.. code-block:: python

   [BATCH_WARPING] 
   warpedOutImgBaseName     = imgWarpedByDVF_          # prefix of the transformed output images
   outDir                   = <Path/to/where/the/results/of/the/bath-warping/step/will/be/written/>
   dvfPostFix               = __dvf_dvfCor_Nto1.nii.gz  # This usually does not need to be changed 
   
   # Optional, comment if not using
   scaleLungIntensity       = 1            # Set to 1 if you would like the lung intensity to change according to the Jacobian of the deformation
   referenceImgName         = <Complete/path/to/the/image/that/you/would/like/to/warp.nii.gz>
   lungLowerThreshold       = -760         # If using HU values this should be a reasonable lower threshold for lung tissue
   lungUpperThreshold       = -730         # If using HU values this should be a reasonable upper threshold for lung tissue
   additionalResampleParams = -pad -1000.0 # Set the padding value of the resampling to -1000


.. note:: 

  The parameter ``dvfPostFix`` usually does not need to be changed. You should get a list of all post-processed DVF files by doing an ``ls [BATCH_POSTPROCESSING][outDir]*[BATCH_WARPING][dvfPostFix]`` on the command line. Replace the elements in square brackets with the parameters in the corresponding sections of your configuration file.



General notes
-------------

* Image files are always saved in nifti image format. To save space it is highly recommended to use the compressed *g-zipped* file format. This can be done by selecting the file extension ``.nii.gz`` instead of ``.nii``.

* Each processing step has its own output directory

* Some computations in the later parts of the processing pipeline depend on previous results. Some results are thus expected in the output directories. 

* End parameters that define a path with the separator /

* Please use the forward slash / as a path separator also on Windows systems



.. _config-file:

The configuration file
----------------------

The contents of the complete configuration file is given below. 

.. literalinclude:: ../../PyCidX/XCAT_dvf_processing.cfg
   :language: python


Other (potentially useful) functions
------------------------------------

The functionality described below is used as part of the post-processing framework. However, some functions may be
useful as on its own.

Converting XCAT binary output to nifti images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The XCAT binary images can be converted to nifti image format by calling file ``convertXCATBinaryFile.py`` from the
command line alongside with some parameters that are required to fill in the nifti header information correctly. When
run without any parameters, it will produce the following output:

.. code-block:: python
   python convertXCATBinaryFile.py
   >> Tool to convert XCAT binary image files into nifti images.
   >> Output: outputFileBaseName   Nifti image with the structure.
   >> Usage: convertXCATDVFTextFile.py pathXCATDVFFile outputDir nx ny nz dx dy dz
   >>    - pathXCATBinFile    -> Path to the .bin file output by XCAT
   >>    - outputDir          -> Path to which the output nifti image will be written. Gets the same name as the XCAT file
   >>    - nx ny nz           -> Number of voxels in x, y, and z direction (optional, defaults to 512 x 512 x 151)
   >>    - dx dy dz           -> Voxel spacing in x, y, and z direction (optional, defaults to 500/512 x 500/512 x 2)
   >>    - convertToHU        -> If converting an attenuation map, if the conversion to HU values should be performed set this to 1 [0]

Define:

* the input image (.bin XCAT attenuation),
* the directory to where you would like to save the nifti image - the file name will be the same as the .bin file,
* size (nx, ny, and nz) and
* spacing (dx, dy, and dz).
* (optional) 0/1 if you would like the image intensities to be converted into HU values.

.. note::
   Note that the image will have the origin set to (0,0,0).


Converting XCAT text deformation vector fields to nifti images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The XCAT software writes the output DVF to a text file. This package contains a python script to convert this into
a nifit image file that can be used with niftiy-reg for instance: ``convertXCATDVFTextFile.py``.

When the script is called without parameters, the following help message will appear:

.. code-block:: python

   python convertXCATDVFTextFile.py
   >> Tool to convert XCAT DVF text output files into nifti images.
   >>   Output: outputDir/XCAT_outFileName__dvf.nii.gz       DVF image with a displacement vector (in voxels).
   >>
   >> Usage: convertXCATDVFTextFile.py pathXCATDVFFile pathToNiftiImage outputDir nx ny nz dx dy dz
   >>         - pathXCATDVFFile       -> Path to the DVF text file output by XCAT
   >>         - pathToNiftiImage      -> Path to an image showing the XCAT anatomy in nifti image format
   >>         - outputDir             -> Path to where the output will be saved
   >>         - nx ny nz              -> Number of voxels in x, y, and z direction (optional, defaults to 256 x 256 x 161)
   >>         - dx dy dz              -> Voxel spacing in x, y, and z direction (optional, defaults to 2 x 2 x 2)
   >>         - topZ0                 -> Number of z-slices (from superior) in the lung-segmentation that is forced to zero, optional.

Define:

* The DVF text file ans an input,
* A nifti image (CT-like/attenuation) converted as described in the section above
* The number voxels of the original simulation (nx, ny, and nz)
* The voxel spacing (dx, dy, and dz)

.. note::
   Note that when using the generated DVFs with the nifty-reg package, the DVF defines relative displacements ( and not
   absolute deformations).