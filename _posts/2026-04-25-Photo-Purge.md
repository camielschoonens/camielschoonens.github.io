---
layout: post
title: "My photo cleanup workflow"
date: 2026-04-25 15:30
description: "A practical workflow for cleaning up years of family photos using PhotoSweeper and ExifRenamer on macOS. Deduplicating, trimming burst series, and renaming files by EXIF data."
categories: [File Management]
tags: [photography, workflow]
pin: false
toc: false
published: true
---
> This post was originally posted on March 1st, 2026 at [camiel.schoonens.nl](https://camiel.schoonens.nl/2026/the-great-family-photo-purge/).
{: .prompt-info }

I have 25 folders (filled with many subfolders) filled with photos from 2001 to 2026. All the folders are stored on my NAS. When I import new photos from my DSLR into Lightroom, I automatically back up the photos to my NAS drive. For pictures taken on the four iPhones in our household, I back up the images from iCloud to my NAS, usually once a year. The result of this process? There are thousands of JPEG files in these folders, and since 2023 (the year my youngest turned 12), the number of pictures we take has increased exponentially.

I’ve long wanted to clean up this digital mess by removing duplicate photos and selecting one photo from a series of photos (often from the children). The purpose of all this isn’t to save disk space; it’s to create physical photo albums for the years since we have children and are a family.

Yesterday morning, I said to myself, “At least figure out how you are going to approach this as a first step.” Spoiler: I think I did, and I’m documenting my thoughts here for future reference and for others who may deal with the same challenges.

I’ve made use of the following two MacOS apps: PhotoSweeper and ExifRenamer, and tried out this workflow on two annual folders with a combined total of a little over 3000 photos. This is my approach:

1. Create a ‘year’ zip file of every single year folder available for backup purposes. I’ve uploaded these individual zip files to my Backblaze, my off-site S3 backup solution.
2. Copy a single folder with all subfolders from my NAS to my desktop computer, mainly for performance reasons while processing, but also to create a ‘working folder’ without touching the files on my NAS.
3. Have PhotoSweeper analyze the contents of the year folder and subfolders on my desktop. My desktop computer is an M4 Mac Mini, and PhotoSweeper is incredibly fast at this.
4. Manually go through three types of compares PhotoSweeper does on the folder and its subfolders: 1. Filename and exif comparison, 2. Series comparison for images taken within one minute of each other, and 3. Having the app’s algorithm handle image comparison on my behalf by ‘looking’ at the images.
5. Mark all the images I want to delete based on the analysis, following PhotoSweeper’s advice about which one to delete 9 out of 10 times.
6. As I’m working on my desktop, on the copied versions of the photos, I’m letting PhotoSweeper delete all the marked photos for me.
7. I’m now left with a cleaned-up annual folder and still lots of subfolders, but no more duplicate photos or series of photos that I need.
8. The final processing step is to have ExifRenamer go through the folders and all the subfolders and move the individual jpeg images to a new folder, still on my desktop. While ExifRenamer moves the files, it updates the filenames based on the Exif data it reads. I name the files with the following syntax: yyyymmdd hhmmss – (Camera Name).jpg. 
9. As a result, I end up with a cleansed annual archive of images that are sorted from A-Z in the file system. As a bonus, the filename gives me a hint at the image’s quality based on the camera name. I suspect this will come in handy once I start selecting the pictures for my printed photo album.
