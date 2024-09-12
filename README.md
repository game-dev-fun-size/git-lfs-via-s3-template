# git-lfs-via-s3-template
Template project demonstrating a method to use AWS S3 buckets as the backend for git-lfs storage

## Overview
When using git with projects with large binary assets, e.g., Unreal Engine projects with all their `*.uasset` files, those shouldn't be handled the same way as typical code files, and you'll want to use git-lfs to manage those files. However, the git-lfs storage offered through github is limited, and expanding that storage is relatively expensive. In this project template, I'll show how you can set up a repository to use git-lfs and store your large files in an AWS S3 bucket, where storage isn't free, but _far_ less expensive than storing your lfs files on github.

NOTE: The local lfs-server I use here does _not_ support the git-lfs file-locking api (yet).

## Project Setup
### S3 Bucket Setup
1. If you don't already have one, you'll need to create an account with Amazon Web Services (AWS).
2. In AWS, you'll want to create a new bucket in Amazon S3. If you have multiple projects, you can use a single bucket for all your projects; it will need to be organized into directories either way. You should block all public access to this bucket. We'll set up access policies for your project in a minute.
3. Create a nested directory structure for your repository, starting with an `lfs` directory, then the name of your organiation, and then the name of your repository. For example, in my bucket, the files for the `game-dev-fun-size/boss-fight` project are in my bucket's `lfs/game-dev-fun-size/boss-fight` directory. This directory setup is important because the lfs server we'll be using expects it to follow those conventions.
4. Next you'll need to set up an IAM policy to use this directory. The git-lfs server will need permissions to ListObjects on your entire bucket to be able to discover the directory, and to read and write objects in the specific project directory. Here's an example of what the policy JSON for the "Boss Fight" project might look like:
   ```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::git-storage-bucket"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:*Object"
            ],
            "Resource": "arn:aws:s3:::git-storage-bucket/lfs/game-dev-fun-size/boss-fight/*"
        }
    ]
   }
   ```
5. Now that you have a policy set up, you'll want to create IAM users for everyone who will be contributing to the project, and then add that policy directly to each user. I like this approach because I'm working with several short-lived projects each with different team members on it, but depending on your organization, you may want to manage user groups instead.
6. For each user, create an access key and securely share it with that user. They will need to use the access key when running the lfs server that connects to your bucket.
### Git Repository Setup
1. This git repository includes:
    1. A reference to a submodule, https://github.com/jasonwhite/rudolfs, which is a git-lfs server that can run locally and connect to your S3 bucket for storing and retrieving large files. It requires Rust to run, which you can install [here](https://www.rust-lang.org/tools/install).
    2. A `.gitattributes` file that specifies which files should be stored using git-lfs. This is pre-populated with the file extentions for Unreal Engine assets, but you can add things like `.blend` files, video references, `.psd` files; whatever potentially large file types it might make sense to keep in your project repository.
    3. An `.lfsconfig` file, which should be updated with your repository's organization and repo name to match the folders you made in S3.
    4. A `lfs-server.sh.template`, which is a shell script for running the gif-lfs server that's only missing AWS credentials.
    5. A `.gitignore` file for `lfs-server.sh`, which is meant to prevent you from accidentally committing the lfs-server-running shell script with your AWS credentials in it!
2. You should update the file types in your `.gitattributes` file to reflect the needs of your project, update the `.lfsconfig` file with your project's org and repo name, update `lfs-server.sh.template` to use your bucket name and create a copy of `lfs-server.sh.template` with `.template` stripped off and your AWS credentials in place.
3. Using a terminal in your repo's root folder (I use Git Bash), run `./lfs-server.sh` to start the lfs server and ensure that it can connect to your S3 bucket.
4. As long as the lfs server is running in the background, you can now add, commit and push files to your repository, and the files types defined in `.gitattributes` will be routed to your S3 bucket via the lfs-server.

***
Once your bucket, repository and game (or whatever else you're up to) project are set up, you can update the README to include these instructions for new users:

### Prerequisites
- You'll need to have [Rust installed on your machine](https://www.rust-lang.org/tools/install) to run the local git-lfs server.
- [Install git](https://git-scm.com/downloads) if you haven't already and you'll need to install git-lfs on your machine as well: `git lfs install`
- This project runs Unreal Engine 5.3, so [you'll need that installed](https://www.unrealengine.com/en-US/download) to open the project.

### Getting Started
- Run `$ export GIT_LFS_SKIP_SMUDGE=1` to prevent git from trying to download lfs files when cloning
- Run `git clone --recursive https://github.com/YOUR-ORG-HERE/YOUR-REPO-HERE.git` to clone the repository (including a submodule that implements a local git-lfs server)
- Open lfs-server.sh.template, add your AWS credentials for the project's lfs bucket, and then *save as* lfs-server.sh (this file is on the .gitignore to prevent you from accidentally commiting your credentials.)
- From within the project root, run `$ ./lfs-server.sh` to start the local lfs server, "Rudolfs". This will need to be running any tme you pull from or push to the repo to sync binary files to the s3 bucket.
- Run `$ export GIT_LFS_SKIP_SMUDGE=0` to tell git to automatically pull and clone lfs files from now on
- Run `$ git lfs pull` to download the missing lfs files

### Notes
- Depending on your internet speed, some files may take longer to upload/download and you'll get errors trying to push/pull those files unless you extend the lfs activity timeout (the default is 30sec): `$ git config lfs.activitytimeout 120`
- On my setup I had to limit git-lfs to only upload one file at a time: `$ git config --global lfs.concurrenttransfers 1`. This makes it less likely to time out when uploading to S3 -- if it's trying to upload 5 big files at once and it takes more than two minutes then it fails entirely and just keeps retrying and failing.
  
