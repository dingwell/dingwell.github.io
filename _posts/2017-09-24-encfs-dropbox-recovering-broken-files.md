---
layout: post
title: "EncFS, Dropbox and broken files"
date: 2017-08-15
categories: coding
tags: 
---
{::comment}
The filename must be on this format:
2010-01-01-some-sort-of-title.md
the title in the header will be the one displayed on the web pag.
{:/comment}

If you've used EncFS together with a file-syncing service,
you might have ended up with corrupt files that are difficult to recover.
If the encrypted file becomes corrupt for some reason,
it can be very hard to recover your data.
Some services allow you to revert your files to a previous stage,
in case something goes wrong
but that is of little help if your entire directory tree is encrypted.
In these situations it is useful to be able to translate decrypted paths
to their corresponding encrypted paths,
and that's what this post is all about.

## Disclaimer
What I describe here is in no way a substitute for proper incremental backups
of your data.
Good backup practice will solve these problems altogether.
I use this technique to recover corrupted files as a first attempt
when I do not have direct access to my backups.

## The problem
I've found that a common case with Dropbox is caused by conflicting copies.
When Dropbox encounters conflicting copies of a file,
it move one of the files to a copy named
"{original file} ({USER}'s conflicted copy {date})"
A convenient solution to the problem,
unless you're running EncFS, that is...

With EncFS, this has happened to me a lot: 
The conflicting copy will be saved as:
  "{encrypted_name} ({USER}'s conflicted copy {date})"
The corrupt file is usually named:
  "{encrypted_name}"

EncFS knows how to handle the latter
and generates a corresponding file when mounting the directory.
But that does little to help us when the file is corrupt.
EncFS cannot interpret the filename of the conflicted copy, 
and therefore, it does not appear in the mounted directory.

In order to replace the corrupt file with the (hopefully) intact
conflicting copy, you need to known the encrypted directory path.

This is where 
[encfsctl](https://linux.die.net/man/1/encfsctl)
can be of help,
You can use encfsctl to find the full path of the encrypted file
if given the configuration file of the encrypted directory
as well as the path to the mount point and the decrypted corrupt file.

But it is rather tedious to use every time you need to fix something
(I always need to flip through the man pages)

I wrote a script give me all information needed for a specific stash using a short command:

~~~
./get_encfs_fname.sh plans/to_dominate/the_world/2020-11-11/doomsday.py               #relative path
./get_encfs_fname.sh /mnt/secret/plans/to_dominate/the_world/2020-11-11/doomsday.py   #absolute path
~~~

This can neatly be integrated in e.g. thunar for quick access
when browsing your files.
The output will look something like this:

The output looks something like this:

~~~
Encrypted file should be:
T9PdY6pm48EwPeEdwUTt/Z2ZlKKLF8HbbCqkdE,rgM8TNP/Qq203eQx7OO1WakWbUvaRqdwm/WigPMtUcCMVXHgExrK,TzwMaT/6J9j1Z8-5dtG-gy,y4q,0c2hj

Absolute path (without file name):
/your/home/Dropbox/my_secret_encrypted_folder/T9PdY6pm48EwPeEdwUTt/Z2ZlKKLF8HbbCqkdE,rgM8TNP/Qq203eQx7OO1WakWbUvaRqdwm/WigPMtUcCMVXHgExrK,TzwMaT/

Directory structure:
/your/home/Dropbox/my_sectred_encrypted_folder
┌──────────────────┘
└── plans :: T9PdY6pm48EwPeEdwUTt
    └── to_dominate :: Z2ZlKKLF8HbbCqkdE,rgM8TNP
        └── the_world:: Qq203eQx7OO1WakWbUvaRqdwm
            └── 2020-11-11 :: WigPMtUcCMVXHgExrK,TzwMaT
                └── doomsday.py :: 6J9j1Z8-5dtG-gy,y4q,0c2hj
~~~

This is the script:

~~~ bash
#!/bin/bash
# Returns the file name and path of the encfs6 encrypted file

ENCFS_ROOT="/your/home/Dropbox/my_secret_encrypted_folder"
MOUNT_POINT="/mnt/secret"          # No trailing '/'
export ENCFS6_CONFIG="$ENCFS_ROOT/.encfs6.xml"

FNAME_DEC=$1


print_root_path(){
  # For a given root path, eg: /foo/bar/etc
  # Will print:
  #   /foo/bar/etc        // line 1
  #   ┌────────┘          // line 2
  LINE1="$1"

  # Remove first character from path
  #NO_LEAD=${LINE1:1}
  # Remove the last directory/file from path:
  #NO_LEAD_END=$(echo "$NO_LEAD"|sed -r 's;/[^/]+$;;')
  LINE1_NO_END=$(echo "$LINE1"|sed -r 's;/[^/]+$;;')
  
  # Now we can generate the back-trailing tree branch thingie (line 2):
  # Do this replacing each character with '─' and wrapping the whole line
  # with '┌' and '┘':
  LINE2="┌$(echo $LINE1_NO_END|sed -r 's/[^─]/─/g')┘"

  echo "$LINE1"
  echo "$LINE2"

  return 0
}

print_tree_pair(){
  export IFS="/"  # delimiter for splitting path into array of dirs
  read -ra T1 <<< "$1"  # Tree one, eg: encrypted path
  read -ra T2 <<< "$2"  # Tree two, eg: mounted path

  SPC=""
  BRANCH="└──"
  for ((i=0;i<${#T1[@]};++i)); do
    echo "$SPC$BRANCH ${T1[i]} :: ${T2[i]}"
    SPC="$SPC    "
  done
  return 0
}

if [[ ! -f $FNAME_DEC ]]; then
  echo "ERROR: '$FNAME_DEC' is not a valid file!" 1>&2
  exit 1
fi

# If $FNAME_DEC is an absolute path, try to remove the leading (mount point)
# directories from it before proceeding
if [[ $FNAME_DEC == '/'* ]]; then
  echo "The given path is an absolute path."
  FNAME_DEC=$(echo "$FNAME_DEC"|sed -r 's;^'"$MOUNT_POINT/"';;')
  echo "The relative path should be: '$FNAME_DEC'"
fi

FNAME_ENC=$(encfsctl encode "$MOUNT_POINT" "$FNAME_DEC")

echo "Encrypted file should be:"
echo "$FNAME_ENC"
echo ""
echo "Absolute path (without file name):"
echo "$(dirname $ENCFS_ROOT/$FNAME_ENC)"
echo ""
echo "Directory structure:"
print_root_path "$ENCFS_ROOT"
print_tree_pair "$FNAME_DEC" "$FNAME_ENC"

echo "Press enter to continue"
read TMP
~~~


Note that encfsctl will not actually check that your password is valid.
If the returned directory tree does not match that on your system,
then you have probably mistyped your password.

## A word of warning
If you care about your privacy enough to be using EncFS,
then you probably already know this but just to be on the safe side:
[EncFS has been shown to be (somewhat) unsecure](https://defuse.ca/audits/encfs.htm).

There is a project aimed at countering this problem, called
[CryFS](https://www.cryfs.org/).
CryFS will bundle your data into blocks of fixed size.
The file syncing program will be able to see the blocks but not the individual files.
This approach allows you to retain some of the benefits of encfs
(you only need to sync part of your encrypted directory when something changes)
but with improved security.
A problem with CryFS is that if a block fails to sync,
you might end up with numerous corrupt files.
As far as I know there is no quick fix for that;
unless, of course,
you do things properly and run local backups of your decrypted files!
(Also, CryFS is still in BETA so, yeah, try at your own risk...)
