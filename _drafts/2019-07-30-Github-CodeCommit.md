---
layout: post
title: How to clone Github repos to CodeCommit using Lambda
summary: Regular backup of my Github repos to CodeCommit using Lambda, Lambda Layers and SAM
tags: [aws, codecommit, lambda, ]
---

### Backup your backups
I've always had backups of my stuff, first it was photos on multiple hard drives in different locations, then we had online backup services and then we had the cloud!

I still keep copies of that stuff across different locations, but one thing that was bugging me was that I didn't have a backup of my Github repos.

Now lets face it the world would probably be a better place without some of my code hacks lying around, but this is the place where I keep and host this blog. Again, some may argue that loosing this wouldn't be a bad thing either ;) But I've spent a lot of time writing here, I enjoy it and it's great to see if and how I am progressing. So it would be a shame to loose all that.

This blog has also been about learning new things and documenting as I go, so sometimes I pick the road less travelled in an effort to keep learning. So yes, the easy option would have been to configure my profile to have multiple remotes and push to both of them at the same time.

I wanted a challenge so came up with the idea of getting Lambda to poll my account, clone the repos and push them to CodeCommit.

### Gitops

I know about 6 Git commands off the top of my *head* (ha ha Dad joke) and they can get me out of trouble with the odd rogue commit, but most of the time I'm using VS Code and the built in Git UI. So I guess I went into this a bit naive, but ended up doing a bit of research and found [GitPython](https://gitpython.readthedocs.io/en/stable/tutorial.html) which unsurprisingly is a Python wrapper for running Git commands.

A few things to note about GitPython
* You still need git! Yes that's write it's just a wrapper and you still need Git installed
* You need Python 3! This tripped me up as I found another helper for CodeCommit that was in version 2 and gave up trying to port that.

So the actual Python code wasn't too complex, I get a list of my repos from a JSON file I'm hosting...on Github, then clone these to the local storage on Lambda and push them to CodeCommit. That's right Lambda has some local storage you can use for this available at '/tmp'

This the Lambda entry function
```python
def run(event, context):
    #Get a list of repos as JSON
    repoList = requests.get('https://raw.githubusercontent.com/msimpsonnz/gitbackup/master/repo.json').json()
    #Run over each repo
    for repoName in repoList["repos"]:
        print(repoName)
        clone(repoName)
```

Then we clone each repo, check CodeCommit, create a repo if there isn't one, then setup the remote and then the final push:
```python
def clone(repoName):
    localDir=f'/tmp/{repoName}'
    try:
        shutil.rmtree(localDir)
    except:
        pass
    repoUrl = f"https://github.com/msimpsonnz/{repoName}.git"
    localRepo = Repo.clone_from(repoUrl, localDir)
    print(f'Cloned repo: {repoName}')
    remoteRepo = getOrMakeRepo(repoName)
    print("Created remote repo")
    remote = localRepo.create_remote(name=remoteRepo['repositoryName'], url=remoteRepo['cloneUrlHttp'])
    print('Created remote')
    remote.push(refspec='master:master')
    print('Pushed to master')
```

Simple enough right? Wrong!

### Layers and layers of fun

If you run the above code as is, you will get an error `Failed to initialize: Bad git executable.` Yep that is correct, we need Git installed! We could install it every time the Lambda runs but that is less than ideal so I wanted to play around with [Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) and end up finding someone that was doing CI builds with Lambda and has a [public layer](https://github.com/lambci/git-lambda-layer) with Git installed already!

I'm planning on building my own layer but wanted to get the thing working first.

So a couple of steps in the write direction, I could clone the repo and add the remote. The main issue I faced with this was connecting to CodeCommit, I was getting the following:
`stderr: 'fatal: could not read Username for 'https://git-codecommit.ap-southeast-2.amazonaws.com/v1/repos/misc': No such file or directory'`

### When in doubt add another layer

Now this is a solved problem, CodeCommit will give you Git credentials, you can keep these in Parameter Store or Secrets Manager and you are away, the setup for that is [here](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html)

I wanted to see if I could use the Lambda IAM role to get temporary credentials for CodeCommit. This is how it works on a development machine, you have to update `.gitconfig` to use a special AWS CLI command which goes off and gets a v4 signed URL to use in the background, the details on the developer setup are [here](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html#setting-up-https-unixes-credential-helper).

It is a bit more difficult to do this with Lambda as we have a read only file system and constrained environment. I did find some interesting things out there, specifically [this one](https://cloudbriefly.com/post/connecting-to-aws-codecommit-repositories-from-an-aws-lambda-function/) which was using a Python script (version 2!!) to replace the CLI credential helper. I spent a bit of time trying to port this to v3 but in the end it would not play nice with GitPython.

So I decided to focus on replicating my local environment and use the AWS CLI as the Git credential helper. This nice thing here is there is already a Server Application Repository (SAR) for the CLI running in a Lambda Layer, you can find that [here](https://github.com/aws-samples/aws-lambda-layer-awscli).

Once I had the CLI available, I obviously configured my Lambda Role to have CodeCommit access, then update the `.gitconfig` file as part of my Lambda deployment as follows:

```config
[credential]
    helper = !/opt/awscli/aws codecommit credential-helper $@
    UseHttpPath = true
```

The other key thing was that we need to change Lambda $HOME variable so it could pick up the new `.gitconfig` file

```python
os.environ['HOME'] = '/var/task'
```

Once that was done we were on the home stretch.

### SAM SAM

I am a big fan of the AWS Cloud Development Kit, see my previous posts, but was keen to use SAM again as I haven't used it in a while.

Nothing ground breaking here, just the standard things and some parameters for the info you need to provide. You should have deployed the AWS CLI Layer from the above link in you account as you will need to provide the ARN for that in the SAM deployment. Below is the Lambda resource from the template:

```yaml
Resources:
  GitBackup:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: git-backup
      Handler: handler.run
      Runtime: python3.7
      MemorySize: 1024
      Timeout: 300
      Policies:
        - AWSCodeCommitPowerUser
      Layers:
        - arn:aws:lambda:ap-southeast-2:553035198032:layer:git:5
        - !Ref LayerAWSCLI
      Events:
        GitBackupScheduledEvent:
          Type: Schedule
          Properties:
            Schedule: cron(0 18 ? * FRI *)
```

That's it for now! Check out the full repo [here](https://github.com/msimpsonnz/gitbackup). I'm going to clean it up and see if it can be triggered by incoming commits to Github, but that is for another day...night.