---
title: "GSoC'19 - GVfs and the Google Backend demystified"
toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true
categories:
    - GSoC 2019
tags:
    - GSoC
    - Open Source
    - GNOME
    - GVfs
    - libgdata
    - meson
---

## Introduction

Note: Due to time limitations, I haven't been able to devote much time to writing a blog post. Each time I started, some or the other thing bothered me and I ended up having a draft. My humble apologies to my readers.

So, over the past 3 months or so, I've been working on the Google Backend for GVfs (GNOME Virtual File System), and as of today, the backend is in a state where it's completely useable. Earlier, a large number of operations were disabled. So, if you tried to copy a file from one folder to the other, you'd be given an error "Operation not supported". Now, you may be wondering what's there in a simple copy operation that the developers/maintainers can't fix, or shouldn't something like Google Drive backend for GVfs receive better attention since a great deal of peope keep their important data on their G-Drive?

The answer isn't a yes or no, and it's much more subjective since it pertains to the state of current open-source software. One of the big reasons has been that OSS always lacks man-power, and that the problem at hand wasn't trivial in any sense. My mentor (Ondrej Holy), is the sole maintainer of a project as big as GVfs, and he certainly doesn't have the time of look at each backend's issues.

## Some Preliminary Information

Google Drive is a database-based system - this means that each file/folder in Drive is identified uniquely by an ID. This ID is automatically generated by Drive API each time we upload/create a new file. These IDs are inherently non-human readable, and to support nicely-readable titles, Google Drive stores the title of a file in the file metadata. This is quite different from POSIX world where the title of the file/folder inside a parent serves as the unique identifier of that particular file/folder.

As a result, from the perspective of a software, titles are completely meaningless on Google Drive except for the human readability. So, if we try to create multiple files having same title inside same parent on Google Drive (either web of using API), it's completely allowed and you may eventually have a mess with multiple files having different content with all of them having same titles.

Let's adopt some convention for what's following -- any `entry` mentioned below means a [`GDataDocumentsEntry`](https://developer.gnome.org/gdata/0.17/GDataDocumentsEntry.html) object (i.e. a file or a folder). Moreover, for hash tables, `(a, b) -> c` means that tuple `(a, b)` acts like a key and the corresponding value is `c`.

## The ABCs of Volatile-Paths

The first thing which may have propped up in your mind is WTH are volatile paths? But before understanding that, we need to look at what all data structures the Google Backend uses to keep track of files. At the root of `GVfsGoogleBackend` we have 3 main structures, and 1 auxiliary structure. We'll talk about the main structures first, and the auxiliary one will be explained once we understand what volatile paths are. They are as follows:

* **entries** (type `GHashTable *`)**:** A mapping from the ID of an `entry` (file/folder) to the corresponding `GDataEntry` pointer. It acts like a fast store to quickly fetch a `GDataEntry` just by looking at the ID.

* **dir_entries** (type `GHashTable *`)**:** A mapping from a tuple `(a, b) -> c`. Here, `a` can be either the title or the ID of a child entry inside a parent having the ID `b`. This maps the children of a directory to their corresponding `GDataEntry` objects.

* **dir_timestamps** (type `GHashTable *`)**:** A mapping from the ID of an `entry` to the last time the entry was fetched from Drive API. This is used to check whether the entry has gone stale or not. Currently, the condition for an entry being stale is that it needs to be atleast 60 seconds old.

* **dir_collisions** (type `GList *` (element `GDataEntry`))

So, back to our question, what are volatile paths?

Normally in POSIX systems paths are expressed as `/folder1/folder2/foo`. Now, since titles are totally deceptive in Google Drive, for each file, we create a fake path for the file using the IDs. Hence, if I have the ID of `folder1` as `id1`, ID of `folder2` as `id2` and ID of `foo` as `id3`, then our actual path (for Google Drive backend) would be `/id1/id2/id3`. But GIO allows a file to be addressed using the display\_name (or title) as well (using functions like [`g_file_get_child_for_display_name()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-get-child-for-display-name)). So, the file `foo` can also be accessed using the "fake path" `/id1/id2/foo`. This is a volatile-path, and it has to be maintained recursively for all the directories/files.

Such paths don't exist anywhere in Google Drive because any file (say `foo`) can be fetched just by its ID `id3`. Since, GVfs is an extension point for GIO (and it expects POSIX-like paths), we maintain these actual and volatile paths in Google Backend using the above 4 data structures. Before going further, I'd like to point out that there's this [excellent blog post](https://debarshiray.wordpress.com/2015/09/13/google-drive-and-gnome-what-is-a-volatile-path/) by Debarshi Ray on Volatile-paths. It'll give you a good insight on volatile-paths.

## What's the deal with Volatile-paths?

For understanding the main issue, we need to firstly look at how clients using GIO implement various operations. Let's take for example, the rename operation for which GIO provides [`g_file_set_display_name()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-set-display-name). After this call finishes, normally clients like nautilus and the `gio` commandline tool calls [`g_file_query_info()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-query-info) on the renamed title to check whether this file has changed or not. Now, for rename operation, there isn't much of an issue because `g_file_query_info()` gets called on the `new_title`, which will get resolved because of the volatile-path `path_to_parent_of_file/new_title`.

But lets think for a moment about copy/move operations. The way nautilus operates is that it calls [`g_file_copy()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-copy) from source to destination file (a `GFile` object). So, for a file with actual path `/folder1_ID/file1_ID` being copied from `folder1` to `folder2`, it expects that the final path of the file will be `/folder2_ID/file1_ID`.
But, the ID creation in Google Drive is completely upto Drive API and the actual path of the copied file will be `/folder2_ID/new_ID`. So, what happens is that nautilus expects  for some file on the path `/folder1_ID/file1_ID` which doesn't exist and subsequently calls [`g_file_query_info()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-query-info). Hence, this call fails with a `G_IO_ERROR_NOT_FOUND` and nautilus assumes that the file wasn't copied and shows an error to user.

But, during the copy operation a request to the Drive API had been made asking for a file to be created. This newly created file will have an ID `new_ID`. Hence, even though nautilus gave an error, since the HTTP request has been completed in the background, the file will be created with the title (display name) set to `file1_ID` and this new file having the actual ID `new_ID`.

As a result of this weird behaviour (i.e. loss of titles while copying), the copy operation was completely disabled when copying across different parents in [this issue](https://bugzilla.gnome.org/show_bug.cgi?id=755701) and used to err-out with "Operation Not Supported".

<img src="/assets/images/op-unsupported-err.png" alt="Operation unsupported error" style="display: block; margin: auto;">

Also, since the move vfunc didn't even exist, GIO tried to use (copy + delete) fallback but since copy was disabled, so move would fail too.

## A tale of multiple parents

Have you ever used sym-links? If you've used linux, you probably use them every day when executing a binary (say `python`). These symlinks kind of act like files, but actually contain no data in themselves. What they do is point to some other file having the actual content (can be binary or anything). So, if I were to open a symlink in `vim` and make some changes to its contents, the original parent and all its other symlinks will reflect the changes. Also, if I were to remove the parent file (to which a symlink was created), the link would actually become orphaned (or dangling or dead).

Google Drive also has a similar thing i.e. files having multiple parents. This isn't exactly like a symlink but the ideas are quite same. So, what you can do in Google Drive is to have a file having multiple parents. Each of these files have the same "ID" just that they're shown to be in different folders. Making changes to any of the file reflects in all other links to this file in all othe folders. The sligh difference is that if and when the parent file gets deleted, all the links instead of becoming orphaned get deleted too. One can create these files with multiple parents on the Drive Web UI by holding the `Ctrl` key and performing a drag and drop to another folder.

## An approach to the problem

Our approach to the problem was to store a "volatile entry" to the source file in the metadata of the new file while copying. Google Drive API allows addition of custom "properties"  to file objects in what's called ["Properties Resource"](https://developers.google.com/drive/api/v2/reference/properties). But the problem was that libgdata API didn't support adding custom properties to `GDataEntry` or `GDataDocumentsEntry` types. So, the first thing we needed to do was to extend the libgdata API and add the necessary types. After much deliberation, we decided we'd name this new "Properties Resource" as `GDataDocumentsProperty`, and that it'd be subclassed directly from `GDataParsable`.

I worked upon the creation of this new type along with its integration with `GDataDocumentsEntry`. So, we store all the "Properties Resources" of a file as a `GList` of `GDataDocumentsProperty`. A [MR was opened](https://gitlab.gnome.org/GNOME/libgdata/merge_requests/7) for the same which has now been merged. This was the easier part of a much more complex problem which me and my mentor had addressed successfully. The complex part of integrating this API with Google Backend to support copy/move operation still remained.

The idea was that whenever we tried to copy a file (say `file1` with path `/id1`)  to some other directory `/folder1_ID/new_ID`, GIO would call `g_file_query_info()` on the path `/folder1_ID/id1` and would fail subsequently because such a file just doesn't exist. Our plan was to add this "fake entry" pointing to the `GDataEntry` of the newly copied file. This "fake entry" was `(id1, folder1_ID) -> GDataEntry` mapping. Due to addition of this fake entry to `dir_entries` hash table, our `g_file_query_info`  call wouldn't fail and so our copy operation would succeed.

But then this "fake entry" could also get treated as some other file having ID `id1` and residing in a foler with ID `folder1_ID`. So, while enumerating the `folder1_ID` directory, we'd have to eliminate this "fake entry". What if someone actually wanted to create a new file with its display name set to `id1` inside the `folder1_ID` directory ? A collision. What if I happen to copy a file to another directory and my friend creates a new file having same title on the Web UI? Another collision. What happens if I already have a file with multiple parents, and that the source file is the instance of same file and I try to copy/move? Multiple collisions.

<!-- Remember there was a `dir_collisions` GList? Yes, you're right. We're gonna handle the collisions using that list. But how? -->

Now this is just the tip of the ice-berg and I can't possibly explain all the complications (and corner cases) that arise because of this design. Our design certainly wasn't wrong, but we were trying to link a database-backed system to a POSIX based, and we knew it was gonna be a little cumbersome, but never in our wildest dreams had me or my mentor imagined of such corner cases.

## The plethora of cases, and some weird crashes!

Hopefully, by now you've an idea as to how complex the whole logic is going to be when the cases itself are so weird. You may be surprised, but we actually maintain strict constraints on how objects are pushed/removed from one data structure to the other in cases of collisions of entries. For the simplest case of having a file without any collision, we have the `ref_count` of the underlying GDataEntry will be exactly 3 - 1 for insertion into `entries`, and 2 for insertions into `dir_entries` each for the `(real_ID, parent_ID) -> GDataEntry` and `(title, parent_ID) -> GDataEntry` mappings.

For files colliding on the title of a file (i.e. two files having the same title in the same parent directory), we hold a reference of the corresponding `GDataEntry` in the `dir_collisions` list while the `(actual_ID, parent_ID) -> GDataEntry` mappings as it is.

For files having multiple parents, inserting one of the entry into the cache corresponds to inserting entries for that file in all the parents. This is because the metadata of a file contains a `parents` list which has information for all the parents of a file. This further complicates things because if I haven't visited (or called `enumerate`) on that directory, I just partially know the contents of that directory. If at a later time I perform enumeration on this directory and I get another entry with same title, I'll have to kick out my older entry from `dir_entries` into `dir_collisions`. What if I wanted to remove the entry which was inserted earlier by its title? I simply can't because doing `gio remove /folder1/some_title` would remove the latest entry which corresponds to the title `some_title`. Pretty bad!

Fortunately, If I'm interacting with Google Drive in nautilus (which most of us will do), I can know which entry I wish to remove by simply looking at their last modified times or the contents. Under the hood, nautilus will do the heavy lifting of calling [`g_file_delete()`](https://developer.gnome.org/gio/stable/GFile.html#g-file-delete) by resolving the file using its actual ID instead of the title.

One particularly notorious case I encountered while testing was that of completely losing a file while performing a move operation. It occurs when we do `gio move /folder1_ID/file1_ID ../folder1_ID`. What it does under the hood is a little interesting too. Normally, what happens in such cases is that you get a popup box asking you to overwrite the same file. Google backend doesn't support overwriting files via `G_FILE_COPY_OVERWRITE` flag. So, in these cases, earlier we used to return `G_IO_ERROR_NOT_SUPPORTED` but the catch here is that returning `_NOT_SUPPORTED` prompts GIO to use some other fallback because it makes it think as though the move operation wasn't implemented yet. So, it uses a copy + delete fallback instead. The issue is that doing a copy + delete operation means that I make a backup of the source file, perform the copy operation (it actually does replace + write here) and then delete the source file. But since we don't support the backup option either, the copy + delete results in the replace + write being performed on the same file as source, and deletion of the source file. Indeed, quite messy! :-P

The relevant source code which exists copy and move vfunc is as follows:

```c
if (flags & G_FILE_COPY_OVERWRITE)
{
  /* We don't support overwrites, so we don't need to care
   * about G_IO_ERROR_IS_DIRECTORY and G_IO_ERROR_WOULD_MERGE.
   *
   * Moreover, we don't need to take care of G_IO_ERROR_WOULD_RECURSE
   * too since changing a Folder's parent on Drive is a matter of
   * simply changing "parents" property. We don't have to do any
   * recursive move.
   *
   * Also, the reason why we're returning G_IO_ERROR_FAILED here
   * instead of G_IO_ERROR_NOT_SUPPORTED is because _NOT_SUPPORTED
   * implies that this operation isn't implemented and it tells GIO
   * to use some other fallback for it.
   * So, for move operation, it'll use a copy+delete fallback here.
   * Since, we don't support backups, so the fallback will end up
   * removing the source file completely without making any copy, and
   * we lose the file. Hence, we return G_IO_ERROR_FAILED.
   */
  g_vfs_job_failed_literal (G_VFS_JOB (job),
                            G_IO_ERROR,
                            G_IO_ERROR_FAILED,
                            _("Operation not supported"));
  goto out;
}
```

I'm assuming that most of the people keep their important or frequently accessed things on their Google Drive (atleast I keep all my personal and academic documents on my Drive). Crashes could mean losing some file's data permanently, but fortunately we've elimininated as many as possible of such corner cases via extensive testing.

## What doesn't work?

If I were to give a gist of it, please don't have the titles (or display names) of your files same as the actual IDs of other files. There are countless cases which can/may arise because of this and we certainly can't handle all of them.

Having written so much, I'd like to copy some code verbatim here which has a very readable comment explaining a single case which won't work. It's related to what names can you keep for creating a new folder. I'm sorry, but it's one of the places where we're just limited by the design, and like I said, we can't possibly do it better. Here's the code:

```c
/* This case is particularly troublesome. Normally, when we call
* make_directory, the title supplied by the user or client is used to create
* a file with same title. But strangely, when we're trying to copy a folder
* over to another directory (i.e. to a different parent), the copy operation
* would return G_IO_ERROR_WOULD_RECURSE. This means that nautilus will
* manually create the folders (using make_directory) and copy each file.
*
* The major issue here is that nautilus calls make_directory with the title
* of folder set to ID of the original folder. Suppose, we had a folder "folder1"
* which needs to be copied. Now, copy operation would return
* G_IO_ERROR_WOULD_RECURSE (if the parents are different), but nautilus will
* call make_directory with ID of "folder1" instead of title of "folder1".
* Furthermore, nautilus calls query_info on the "filename" argument right
* after it has done make_directory hence we need to create a volatile entry
* for the folder too.
*
* Below fix is hack-ish and will fail if we wish to create a folder
* whose title equals the ID of any other folder. */
if ((same_id_entry = g_hash_table_lookup (self->entries, basename)) != NULL)
{
  GDataDocumentsProperty *source_id_property, *parent_id_property;

  source_id_property = gdata_documents_property_new (SOURCE_ID_PROPERTY_KEY);
  gdata_documents_property_set_visibility (source_id_property, GDATA_DOCUMENTS_PROPERTY_VISIBILITY_PRIVATE);
  gdata_documents_property_set_value (source_id_property, basename);

  parent_id_property = gdata_documents_property_new (PARENT_ID_PROPERTY_KEY);
  gdata_documents_property_set_visibility (parent_id_property, GDATA_DOCUMENTS_PROPERTY_VISIBILITY_PRIVATE);
  gdata_documents_property_set_value (parent_id_property, gdata_entry_get_id (parent));

  gdata_documents_entry_add_documents_property (GDATA_DOCUMENTS_ENTRY (folder), source_id_property);
  gdata_documents_entry_add_documents_property (GDATA_DOCUMENTS_ENTRY (folder), parent_id_property);

  gdata_entry_set_title (GDATA_ENTRY (folder), gdata_entry_get_title (same_id_entry));

  g_clear_object (&source_id_property);
  g_clear_object (&parent_id_property);
}
else
gdata_entry_set_title (GDATA_ENTRY (folder), basename);
```
## The Road Ahead

I plan on sticking to GNOME organization and will apply for a foundation membership once I feel that I've made enough friends here :-). People here have been amazing, and thanks to Carlos Soriano (@csoriano) and Felipe Borges (@feborges), a total newbie and newcomer like me has had such a great experience! Love you all GNOMEies! <3

As for the Google Backend for GVfs, although I had written in my GSoC Proposal that I'd be writing a full test-suite and possibly get it added to the CI pipeline for GVfs. But like everyone, I had also underestimated the time it takes to produce efficient and fully working piece of software. I've created a basic framework (a simple program) to test the backend. It connects to  google account (via Gnome Online Accounts API) and performs tests on that Drive. It uses `g_test_add_func()` to add test cases and we've planned on covering as many cases as possible.

Thanks to anyone who's come reading this far. You, are worthy of being a knight! :-)

<!-- ## A Humble Thanks -->

<!-- To the founders of GNOME organization for having the vision of having a "complete free software solution for everyone", to all the contributors so far, to the GSoC admin team who believed in me and my qualities and to all the people on #gnome-hackers who've helped me. -->

<!-- A special thanks to my Mentor (Ondrej Holy) for being extremely helpful even after having many personal issues. He's been a true inspiration. Philip Withnall  -->
<!-- You might've heard people in programming fraternity say - "There's no such thing as bug-free software". Combine it with Murphy's law which states that "Anything which can go wrong, will go wrong!" and Boom! -->

<!-- One thing which I'd like to admit is that the problem at hand was much harder than what I had conceived it to be. And neither me nor my mentor had an idea as to how much complex things can get once we delve deeper into the quirks of a database-backend systems like Google. -->


<!-- Google Drive is inherently a database backed system (and arguably the best when it comes to sharing files) meaning that any file is uniquely identified by its ID
   - ### Volatile Paths - A Deep Dive
   - 
   - 
   - ### Copy and Move operation - how POSIX does it and how we hack around it?
   - 
   - ### How can I link my Google Drive to Nautilus?
   - kkk
   - In the settings, there's an option to add a Google account. It asks for login credentials (over web) and seek permissions to a host of Google products (like Calendar, Drive, etc).
   -  -->
