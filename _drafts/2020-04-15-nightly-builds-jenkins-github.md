---
layout: post
title: Nightly Builds in GitHub using Jenkins
---

In this post I share a solution I have introduced for providing nightly builds of the muCommander project on GitHub.


# Why Nightly Builds?

Building and testing your project periodically is a common practice nowadays. Many projects run unit tests before merging pull requests (PRs). Some projects, like [KubeVirt](https://github.com/kubevirt), also run integration tests automatically before merging a PR, while others, like [oVirt](https://github.com/oVirt), run them on demand or after the fact. Various continuous integration tools such as [Travis CI](https://travis-ci.com) and [Circle CI](https://circleci.com) are available for this purpose.  

However, many projects lack periodic releases. The concept of not only building the project but also deliverying unstable releases that are derived from the development branch periodically, possibly on a daily basis in which case they are generally referred to as *nightly builds*, is often missing despite its potential benefit. For users, it is a way to expose new features and by that enable them to provide early feedback. For developers, they may ease validating certain capabilities without the need to compile the code locally and copy the artifacts elsewhere.

# What's the Challenge? 

In some projects, frequent packaging and deliverying of unstable releases may not be worthy due to various reasons. For instance, distributed infrastructure management systems may require relatively high amount of resources and complex installation process, and as such their users may want to avoid deploying unstable releases. As another example, users of mission-critical systems may wish to minimize the amount of changes in their system.  

But I believe the reason for the lack of nightly builds for the majority of projects out there would rather be the lack of a proper place to store them. Nightly builds are clearly useful for most projects, especially for software that is delivered as-a-service (SaaS) or standalone applications, however, it is not easy to find a free place to store them in a way that they could be easily consumed by users.  

Let's look at the muCommander project, for example. Nightly builds, that were built by Jenkins, used to be stored on a local virtual machine. It was then easy to publish them on the project's website. However, the project no longer possesses a local machine that can be available all the time. Today, both our source code and our website are hosted on GitHub (and GitHub Pages) and while GitHub provides a place to store releases, it lacks a mechanism for storing nightly builds.  

# Our Solution

The recently introduced solution for muCommander involves building the nightly builds using a local virtual machine (with Jenkins) that pushes the artifacts to GitHub. This way, the nightly builds are stored  "in the cloud", i.e., on a remote infrastructure that provides better availability than a local machine along with our stable releases that are also stored on GitHub. In addition, the local VM is a safe place to store a token for GitHub that is required for pushing the artifacts. While the local VM may not be running all the time, it can be easily recovered in case of a problem (in the worst case scenario, no new builds are produced but the previous one would still be available).  

I found no integration between Jenkins and GitHub that I could use for pushing the nightly builds to GitHub though. As I have previously mentioned, GitHub does not offer an out-of-the-box mechanism for third party tools like Jenkins or other CI/CD tools for nightly builds. This required me to write the following script that is based on the one I have found on [this post](https://medium.com/@systemglitch/continuous-integration-with-jenkins-and-github-release-814904e20776):

```sh
# Publish on github
echo "Publishing on Github..."
token="<your-token>"
 
# Get the title and the description as separated variables
api_endpoint="https://api.github.com/repos/mucommander/mucommander"
uploads_endpoint="https://uploads.github.com/repos/mucommander/mucommander"
tag="nightly"
name="Nightly"
artifact=$(ls build/distributions)
md5=$(md5sum build/distributions/$artifact | awk '{print $1}')
description="MD5:\n$md5 $artifact"
description=$(echo "$description" | sed -z 's/\n/\\n/g') # Escape line breaks to prevent json parsing problems
 
# Existing release
release=$(curl -XGET $api_endpoint/releases/tags/$tag)
 
# Extract the id of the release from the creation response
id=$(echo "$release" | sed -n -e 's/"id":\ \([0-9]\+\),/\1/p' | head -n 1 | sed 's/[[:blank:]]//g')
 
if [ ! -z "$id" ]; then
    # Deleting the existing release
    curl -XDELETE -H "Authorization:token $token" $api_endpoint/releases/$id
fi
 
# Delete the existing tag
$(curl -XDELETE -H "Authorization:token $token" $api_endpoint/git/refs/tags/$tag) || true
 
# Create a release
release=$(curl -XPOST -H "Authorization:token $token" --data "{\"tag_name\": \"$tag\", \"target_commitish\": \"master\", \"name\": \"$name\", \"body\": \"$description\", \"draft\": false, \"prerelease\": true}" $api_endpoint/releases)
 
# Extract the id of the release from the creation response
id=$(echo "$release" | sed -n -e 's/"id":\ \([0-9]\+\),/\1/p' | head -n 1 | sed 's/[[:blank:]]//g')
 
# Upload the artifact
curl -XPOST -H "Authorization:token $token" -H "Content-Type:application/octet-stream" --data-binary @build/distributions/$artifact $uploads_endpoint/releases/$id/assets?name=$artifact
```

Let's go over this script:  
First, we store our token for GitHub as explained in the abovementioned post.  
Then we initialize some variables. The script makes use of the GitHub API and so the first two variables point to the endpoints of general API calls and upload calls for the `github.com/mucommander/mucommander` repository. The next two variables contain the name of the tag and the name of the release that the nightly build will be associated with. Next two variables contain the name of the artifact and its MD5 hash. Lastly, we set the description of the release to contain the MD5 hash and the name of the artifact.  
Next, we query the existing release that is associated with the aforementioned tag and get its identifier. If the identifier is not empty, it means an existing release of a nightly build exists and it is therefore removed.  
We then remove the existing tag, if it exists, so it will be recreated by the new release on top of the latest commit in the master branch.  
Finally, we create a new release with the abovementioned tag, name and description, and set it as non-draft and pre-release. We then extracts the identifier of the created release and use it to upload the artifact that was previously built by Jenkins to that release.

# What's Next?

With this mechanism, we can start publishing nightly builds in the hope that they will provide us with earlier feedback on the upcoming features in muCommander. The nightly builds would be found [here](https://github.com/mucommander/mucommander/releases/tag/nightly).