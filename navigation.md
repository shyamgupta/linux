[Home](index.md)

# Navigation
---
- `pwd` : Displays the current working directory
- `cd ~` or `cd` will switch to your home directory
- `cd ..` will switch to parent directory (of the current working directory)
- `cd -` will switch to previous directory
- `tree` command is a good way to get a bird’s-eye view of the filesystem tree. Use `tree -d` to view just the directories and to suppress listing file names.
- `ls -a` will list all files, including hidden files.
- **Absolute Pathname** An absolute pathname *begins with the root directory* and follows the tree branch by branch until the path to the desired directory or file is completed.
- **Relative Pathname** A relative pathname *starts from the working directory*. To do this, it uses a couple of special dot notations to represent relative positions in the file system tree. These special notations are `.` and `..` that represent working directory, working directory’s parent directory. 