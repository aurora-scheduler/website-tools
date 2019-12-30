Get the Source Code
-------------------
First things first, you'll need the source! The Aurora source is available from Apache git:

    git clone https://gitbox.apache.org/repos/asf/incubator-aurora

Read the Style Guides
---------------------
Aurora's codebase is primarily Java and Python and conforms to the Twitter Commons styleguides for
both languages.

- [Java Style Guide](https://github.com/twitter/commons/blob/master/src/java/com/twitter/common/styleguide.md)
- [Python Style Guide](https://github.com/twitter/commons/blob/master/src/python/twitter/common/styleguide.md)

Find Something to Do
--------------------
There are issues in [Jira](https://issues.apache.org/jira/browse/AURORA) with the
["newbie" label](https://issues.apache.org/jira/issues/?jql=project%20%3D%20AURORA%20AND%20labels%20%3D%20newbie%20and%20resolution%3Dunresolved)
that are good starting places for new Aurora contributors; pick one of these and dive in! Once
you've got a patch, the next step is to post a review.

Getting your ReviewBoard Account
--------------------------------
Go to https://reviews.apache.org and create an account.

Setting up your ReviewBoard Environment
---------------------------------------
Run `./rbt status`. The first time this runs it will bootstrap and you will be asked to login.
Subsequent runs will cache your login credentials.

Submitting a Patch for Review
-----------------------------
Post a review with `rbt`, fill out the fields in your browser and hit Publish.

    ./rbt post -o

Once you've done this, you probably want to mark the associated Jira issue as Reviewable.

Updating an Existing Review
---------------------------
Incorporate review feedback, make some more commits, update your existing review, fill out the
fields in your browser and hit Publish.

    ./rbt post -o -r <RB_ID>

Merging Your Own Review (Committers)
------------------------------------
Once you have shipits from the right committers, merge your changes in a single commit and mark
the review as submitted. The typical workflow is:

    git checkout master
    git pull origin master
    ./rbt patch -c <RB_ID>  # Verify the automatically-generated commit message looks sane,
                            # editing if necessary.
    git show master         # Verify everything looks sane
    git push origin master
    ./rbt close <RB_ID>

Note that even if you're developing using feature branches you will not use `git merge` - each
commit will be an atomic change accompanied by a ReviewBoard entry.

Merging Someone Else's Review
-----------------------------
Sometimes you'll need to merge someone else's RB. The typical workflow for this is

    git checkout master
    git pull origin master
    ./rbt patch -c <RB_ID>
    git show master  # Verify everything looks sane, author is correct
    git push origin master

Cleaning Up
-----------
Your patch has landed, congratulations! The last thing you'll want to do before moving on to your
next fix is to clean up your Jira and Reviewboard. The former of which should be marked as
"Resolved" while the latter should be marked as "Submitted".
