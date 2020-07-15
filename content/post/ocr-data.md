---
title: "Converting images of tabular data to .csv files"
date: 2020-01-10T11:49:08-06:00
draft: false
---

Have you ever come across a great data source that is unusable because it is contained in an image-based file (e.g. a non-searchable PDF)? Or even worse, data in tables printed on physical paper? This is often a problem for researchers who would like to use data from government or archival sources in statistical analyses. Many solutions for converting image-based tabular data are imperfect, either because the resulting spreadsheet files are poorly formatted or because the text recognition performs poorly (or both).

I have spent considerable time working through this problem and am writing this post to guide other researchers and data scientists through my preferred method: using Amazon Textract. This method requires investing some time at the beginning to get it up and running, and there is a per page charge (except for moderate use during the first year). It also requires some very basic use of Python .[^python] But if you have more than a few pages to convert I think the investment is worth it compared to manually entering in all the data (or using a mediocre solution that requires spending time fixing its output).

This post is organized into three main sections. The first discusses getting hard copy data into digitized image files. The second briefly discusses some options for optical content recognition (OCR). Finally, the third details the process for setting up and using Textract.

# Starting with hard copy data

If you have physical pieces of paper that contain tables of data, the first step is turning them into image files on your computer. Depsite the ease of using a cell phone camera to rapidly capture pages of data, I do not recommend doing so when it is avoidable. Although the quality of these photos can be quite good, the resolution of a photo of a document is often lower than it would be if a scanner were used. Perhaps more importantly, taking photos of documents often introduces spatial distortion (3D page warping and 2D page skew/rotation) that can be difficult to correct post hoc. 

So, if possible, use a document scanner to create image files of the hard copies you have acquired. Set the scanner to the highest resolution possible, especially if the print/text is small. 


## Making the most of cell phone scanning

If using a scanner is impractical due to time or other logistical constraints, you can still successfully digitize hard copy data using a cell phone. But it is very important to pay special attention to the way you take the photos. And, even if you do, you still may be forced to do some pre-processing on the images before conducting the OCR.

First, make sure your camera is set to capture the highest resolution possible and ensure your work area is well lit. Importantly, make sure the documents are as flat as possible when you take pictures of them to minimize warping (text distortion caused by a curving/uneven page surface). Try to position the camera/phone so that the text is perfectly lined up in the photo (lines of text parallel to the top/bottom edges of the photo, vertical lines parallel to the side edges of the photo, etc.). Finally, make sure there are no spots of glare or shadows on the page that would prevent the text from being readable. You should experiment a bit and look at the photos to ensure the text is readable (and run a couple of images through one or more of the conversion methods below). No OCR will be able to recognize text that you can't decipher yourself.

If you do end up with warped images there is one way I have found that dewarps with decent performance. But it is a [Python script](https://github.com/mzucker/page_dewarp/blob/master/page_dewarp.py) written in Python 2.7, which means you would need to install a second Python environment and several packages to use it. It also requires images be rotated  so that warping is horizontal first. Here, an ounce of prevention (better photos) is worth many pounds of cure.


# OCR (optical content recognition) for images of tabular data 

Once you have your images, whether scanned, snapped with a camera/phone, or direct from a government agency, back them up! Especially if you are in the field and accessing the documents in the future would be difficult. The next step is trying out some OCR software options to see how they perform. (I would actually recommend getting your eventual OCR pipeline set up ahead of time so that you can test it with the first few photos you take.) Try to test a variety of page types if there is any variation so that you get realistic results based on your entire sample of pages. For example, some pages might have a mix of tables and text in paragraphs while others only have tables.

There are a few options you can use for the OCR step.[^1] If you have access to Adobe Pro, you can use it to create a PDF of your images and try the Export to Excel feature. Adobe performs OCR on the text and then coerces the content into spreadsheet format and saves it as an Excel file. I have had mixed success with this, but you may find that it performs well for your particular documents. You can also perform just the OCR in Adobe, then use Tabula to convert the OCR'ed PDF to CSV. Again, I've had mixed success with this approach. The main issue I've run into is poorly-formatted spreadsheet output, though the OCR does not always do a great job either.

Another solution is the expensive OCR software [ABBYY FineReader](https://www.abbyy.com/en-us/finereader/), which at the time of this post cost $199 for Windows and $120 for Mac. You can download a free trial to see if this solution works for you, which allows you to process 100 pages within one month before it expires. The software can do simple OCR or OCR to Excel, the latter being what you probably want. ABBYY has worked really well for me on some documents but not on others, so I could not justify paying the steep price.


# The best solution for extracting tables from images: Amazon Textract

The overall best solution to my OCR problems has been the cloud-based [Amazon AWS Textract](https://aws.amazon.com/textract/). It's accuracy has been the best by far in terms of text recognition and table reconstruction. It is not free, though it allows you to process 100 pages with tables for free each month for your first year and additional pages cost only $15/thousand pages. It also requires some setup on the front end, a little bit of command line usage, and some Python/other CS language ability to customize some of how it works. But in my view these costs are well worth it for the accuracy of the CSV output.

Before you dive into Textract, you should upload an image [here](https://console.aws.amazon.com/textract/home?region=us-east-1#/demo) to test its accuracy on your own data (this requires a free AWS account). Once your image is processed, click on the Table tab to see how well it did. Try it again with a few more pages to see if it consistently performs well. 

## AWS setup

If you are satisfied with the test and want to proceed, you'll need to do a few things before you can get Textract up and running at scale (most of these steps are described in the [Textract documentation](https://docs.aws.amazon.com/textract/latest/dg/getting-started.html)). First, [create an AWS account](https://docs.aws.amazon.com/textract/latest/dg/setting-up.html), which is separate from an ordinary Amazon account. You'll need to provide a credit card for any usage above the free tier (100 images per month the first year of your AWS account for Textract). Then, [setup an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html). Next, set up the AWS command line interface (CLI) and SDKs as I describe here. First, download and install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) then [create and download an access key](https://docs.aws.amazon.com/textract/latest/dg/setup-awscli-sdk.html) for the IAM user you previously created. Remember where you saved this file, as you'll need to enter the keys within it in a moment.

Next, you will want to add the folder where AWS was installed to your path. If you are using Windows, go to advanced system settings (can be accessed through the conrtol panel or by searching in the Windows bar). Click "Environment Variables", then "Path", then "Edit". Click "Browse" and select the folder where the aws.exe file is located (likely "C:\Program Files\Amazon\AWSCLIV2"). Click OK on all the open settings windows to save, then close and re-open the command prompt if it is open.

## Python setup

Once you have installed the AWS CLI, it's time to get the SDK set up, which might be a bit more challenging. You'll probably want to use Python if you aren't familiar with the other languages (and if you want to follow along with the rest of the tutorial below). You don't have to learn a ton about Python to do this, but you will have to install it and execute a few simple commands from the command line (so not even "real" python code). Python (and the Boto3 package) will allow you to use Python scripts to communicate with AWS services like Textract. 

I recommend downloading and installing an [Anaconda distribution](https://www.anaconda.com/distribution/) of Python 3.7 (not 2.7), and make sure it's the 64-bit version if you're using a 64-bit Windows machine. Anaconda distributions come with a lot of handy packages pre-installed, especially if you might end up using Python for data science tasks. Once Anaconda is installed, run Anaconda Prompt (in Windows there should be a folder called Anaconda 3 in the Start Menu with the option "Anaconda Prompt (anaconda 3)"). You should see a terminal window open up, in which you can simply type: `pip install boto3` and press Enter. You should eventually see that boto3 was successfully installed. You should then configure the credentials for AWS using [these guidelines](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html#configuration). The access key and secret key are in the .csv file of IAM user credentials you previously downloaded. You should choose an AWS server location close to you (region codes [in the region column here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html). You can leave the default output format blank if prompted.

Back in your Anaconda Prompt window, you should create a new environment for OCR tasks. Working in a specific python environment is often recommended, though I am more of an R person and don't quite understand why... Anyway, type `conda create --name ocr --clone base` to clone the base environment and name the clone "ocr". This is preferable to creating a fresh environment because it copies over all of the packages that are included in the Anaconda distribution, which is one of the main reasons to use Anaconda in the first place. Once the environment is ready, type `activate ocr` to activate it. You shouldn't need to install any other packages for the script I describe below to work, but if for some reason you do need to install additional packages you should do so in the Anaconda prompt after activating the ocr environment. The usual method is entering the command `conda install <package name>`.

## Using Textract

Once you have your Python environment set up, you should download [the python script](https://github.com/evanmorier/ocr/blob/master/textract_csv.py) I adapted for using Textract to ocr images of tabular data and export as .csv. I took an AWS-provided script [found here](https://docs.aws.amazon.com/textract/latest/dg/examples-export-table-csv.html) and made a couple of important improvements.  First, the script will automatically detect PNG and JPG images in the current directory so you don't have to write out every file name. Second, I improved the script so that it can process data with commas in the cells.[^2] To download as a .py file from github, you can right-click on "Raw" and select "Save link as". To use this script, you will either need to add the folder where it is saved to your path, or type the entire path and file name into the console when calling it (e.g. "C:\users\evanp\aws\textract_csv.py").

You should now be just about ready to start Textracting. From the Anaconda Prompt window, navigate to a folder on your computer where you have saved image files to process. This should be a folder that only contains these files, as the script I use will detect and process *all* image files in that folder. I recommend starting with just one image in the folder to test things out. To navigate to this folder in Windows, type `cd "C:\users\..."` (the file path of your folder in quotes). Once there, call the `textract_csv.py` script. One way to do so is by entering the *script's* entire name/filepath into the Anaconda Prompt command line. But if you have added the folder where this script is saved to your path, just should be able to just type `textract_csv.py`.

If all went well, you should soon see a lot of text being output into the Anaconda Prompt. This is the JSON-format data returned by the AWS server (not sure why it spits out into the console but haven't needed to change that yet). Then, a new csv for the image that was processed should appear in the folder where the original image was saved (where you navigated to from within the Anaconda Prompt).

Once all the images in the folder have been processed and you have checked the csvs to make sure they look okay, you should remove the images from the folder so you don't accidentally process them again (i.e. if you add more images to the folder to do another batch). You may also want to keep track of how many images you process each calendar month. AWS gives you 100 free each calendar month for the first year of your account but does not let you track your usage each month.

# Final thoughts

There are a lot of moving parts and things that can go wrong in this process. I don't love python and find that getting it set up almost always causes me some sort of problem. But I'm 100% self-taught in python and I've foudn that googling error messages can often help get you back on track. Let me know if you have issues with any of the steps above and I'll do my best to help out!

[^python]: I think this workflow is doable even if you have never used Python before, and you shouldn't need to actually write any Python code (just get a Python environment set up and use it to execute a Python script). It might take some time to get used to using it (and the command line), though.

[^1]: There are some solutions (like Tabula's PDF to CSV software) that will not work on image files that haven't been OCR'ed first. 

[^2]: The original script treated commas in cells as cell delimiters, producing undesirable results.
