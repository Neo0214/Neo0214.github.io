---
layout: post
title: File Management Simulation - TJU Operating Systems Course Project
author: yik
categories: [ Algorithms ]
image: assets/images/filemanage/cover.jpg
math: true
featured: false
hidden: false
---

## File Management Simulation System Design Report



### Table of Contents

- [File Management Simulation System Design Report](#file-management-simulation-system-design-report)
  - [Table of Contents](#table-of-contents)
  - [Development Environment](#development-environment)
  - [Project Structure](#project-structure)
  - [User Interface](#user-interface)
  - [Operation Instructions](#operation-instructions)
  - [Implemented Features](#implemented-features)
  - [System Analysis](#system-analysis)
    - [Implemented Classes](#implemented-classes)
    - [Explicit Linking Method](#explicit-linking-method)
    - [Bitmap and FAT Table](#bitmap-and-fat-table)
  - [System Implementation](#system-implementation)
    - [Allocating Disk Space for File Content](#allocating-disk-space-for-file-content)
    - [Retrieving File Content](#retrieving-file-content)
    - [Deleting File Content](#deleting-file-content)
    - [Updating File Content](#updating-file-content)
    - [Checking for Duplicate Filenames in the Same Directory](#checking-for-duplicate-filenames-in-the-same-directory)
    - [Saving File Content](#saving-file-content)
    - [Creating Directory Tree](#creating-directory-tree)
    - [Disk File Read and Write Operations](#disk-file-read-and-write-operations)
    - [Directory File Read and Write Operations](#directory-file-read-and-write-operations)



### Development Environment

* Development Environment: Windows 11
* Development Software: PyCharm
* Programming Language: Python 3.11
* Main Imported Modules:
  * PyQt5
  * sys
  * os



### Project Structure

```
FileManageSystem/
    BitMapInfo.txt
    Category.py
    CategoryInfo.txt
    FCB.py
    main.py
    MyDiskInfo.txt
    project_structure.txt
    VirtualDisk.py
    Runtime Screenshot.png
    docs/
        File Management Simulation System Design Report.md
    res/
        cat.jpg
        file.png
        folder.jpg
        icon.jpg
```



### User Interface

![Runtime Screenshot]({{site.baseurl}}/assets/images/filemanage/interface.png){: .mx-auto .d-block}



### Operation Instructions

* Click the buttons on the right side to create new files, folders, or format the disk;
* Right-click on blank areas to create new files or folders;
* Right-click on file or folder names to open or delete files and folders;
* Click on file or folder names to open the file editor or navigate into folders;
* When exiting the file editor, you can choose whether to save changes;
* The upper right section of the interface allows you to enter a name, select a type, and search within the current directory;
* Files and folders can be sorted by file type, filename, and modification date.



### Implemented Features

* Display information about files and folders in the current directory;

* Create and delete files and folders;
* Open folders and edit files;
* Format the disk;
* Search for files and folders;
* Tree-structured directory diagram;
* Sort files and folders by filename, modification time, and file type;
* Navigate to parent directory;
* Display current path.



### System Analysis

#### Implemented Classes

* FCB Class: File Control Block, records filename, file type, modification date, file size, and starting storage location on disk;

* Node Class: Stores child nodes, records parent node, representing relationships between files;

* Category Class: Stores directory information for the entire file system, with root recording the root node. Provides several methods:

  * `free_category(self, p_node)`: Releases the directory of a specified node;

  * `search(self, p_node, file_name, file_type)`: Searches for files or folders under a specified node;

  * `search_in_current_directory(self, p_node, file_name, file_type)`: Searches for files or folders only in the specified directory;

  * `create_file(self, parent_node, fcb)`: Creates files or folders;

  * `check_same_name(self, p_node, name, file_type)`: Checks if a file or folder with the same name exists in the same directory.
* VirtualDisk Class: Simulates disk operations, recording disk size, block size, number of blocks, remaining blocks, memory and bitmap. Provides several methods:
  * `get_block_size(self, size)`: Returns the number of storage blocks required for a specified size;
  * `give_space(self, fcb, content)`: Allocates space for storing content for a specific FCB, writes content to disk, and updates the bitmap;
  * `get_file_content(self,fcb)`: Returns the content stored on disk for a specified FCB;
  * `delete_file_content(self, start, size)`: Deletes content stored on disk at a specified starting position and size;
  * `file_update(self,old_start,old_size,new_fcb,new_content)`: Updates file content.

* MainWindow Class: Main window, maintains primary program logic;
* HelpDialog Class: Help window, displays when the file management system is first opened, providing operational guidance to users;
* CreateDialog Class: New file and folder creation window, displays when creating new files and folders;
* NoteForm Class: Text file editing interface, displays when editing text files;

#### Explicit Linking Method

* In this file system, file storage space management uses the explicit linking method. File content is stored in different blocks on the disk. When creating a file, an appropriate number of free blocks are allocated. When writing to a file, content is written sequentially into corresponding blocks; when deleting a file, the previously occupied positions are simply marked as empty.

#### Bitmap and FAT Table

* Disk free space management is based on an enhanced bitmap design. The FAT table, which stores file location information on the disk, is combined with a traditional bitmap. Empty disk positions are marked with EMPTY = -1, blocks containing files store the position of the next block in the file, and blocks at the end of files are marked with END = -2.



### System Implementation

#### Allocating Disk Space for File Content

```python
 def give_space(self, fcb, content):
        blocks = []
        index = 0
        while index < len(content):
            # Special handling for newlines to ensure '\r\n' is not split
            if content[index:index + 2] == '\r\n' and self.block_size == 2 and index + 2 <= len(content):
                blocks.append(content[index:index + 2])
                index += 2
            elif index + self.block_size <= len(content):
                blocks.append(content[index:index + self.block_size])
                index += self.block_size
            else:
                # Add the last possible block smaller than block_size
                blocks.append(content[index:])
                break

        if not blocks:  # If blocks is empty (content is empty or other cases)
            return True  

        if len(blocks) <= self.remain:
            # Find where to start storing the file
            start = -1
            for i in range(self.block_num):
                if self.bit_map[i] == self.EMPTY:
                    self.remain -= 1
                    start = i
                    fcb.start = i
                    self.memory[i] = blocks[0]
                    break

            if start == -1:  # If no space found
                return False

            # Start storing content from this position onward
            j = 1
            i = start + 1
            while j < len(blocks) and i < self.block_num:
                if self.bit_map[i] == self.EMPTY:
                    self.remain -= 1
                    self.bit_map[start] = i  # Store each piece of data using linking
                    start = i
                    self.memory[i] = blocks[j]
                    j += 1  # Process next block
                i += 1

            if j == len(blocks):
                self.bit_map[start] = self.END  # Mark end of file

            return True
        else:
            return False
```

First, the content is split to ensure text is divided according to the given block size, while ensuring that `\r\n` newline characters are not split across different blocks. Then, the disk is traversed to find the first free storage block, its position is recorded in fcb.start, content is written into it, and the remaining block count is decremented. Next, content continues to be stored from this position onward. In the bitmap, block relationships are stored using linking - each position in the bitmap stores the position of the next block, with the end of file marked by self.END.


#### Retrieving File Content

```python
 def get_file_content(self,fcb):
        if fcb.start == self.EMPTY:
            return ""
        else:
            content = ""
            start = fcb.start
            blocks = self.get_block_size(fcb.size)

            count = 0
            i=start
            while i<self.block_num and count < blocks:
                content += self.memory[i]
                i=self.bit_map[i]
                count+=1

        return content
```

Based on information stored in the bitmap, content from different storage blocks is concatenated together.

#### Deleting File Content

```python
def delete_file_content(self, start, size):
       if start == self.EMPTY or start >= self.block_num:
           return  # If start position is invalid or file is empty, return immediately

       blocks = self.get_block_size(size)

       count = 0
       i = start
       while i < self.block_num and count < blocks:
           next_index = self.bit_map[i]  # Get next index before clearing
           self.memory[i] = ""
           self.bit_map[i] = self.EMPTY
           self.remain += 1

           if next_index == self.END:
               break  # If this was the last block, exit the loop

           i = next_index
           count += 1
```

#### Updating File Content

```python
def file_update(self,old_start,old_size,new_fcb,new_content):
    self.delete_file_content(old_start,old_size)
    return self.give_space(new_fcb,new_content)
```

#### Checking for Duplicate Filenames in the Same Directory

```python
 def check_same_name(self, p_node, name, file_type):
        if p_node is None:
            return True
        # Only check direct children of the given node (parent node)
        for child in p_node.children:
            if child.fcb.file_name == name and child.fcb.file_type == file_type:
                return False  # Found a direct child with the same name and type, return False
        return True  # No file with the same name and type found in the same directory, return True
```

#### Saving File Content

```python
    def save_content(self):
        content = self.textEdit.toPlainText()
        fcb = self.main_form.category.search(self.main_form.current_node, self.filename, FCB.TXTFILE).fcb
        old_size = fcb.size
        new_size = len(content)
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        # Update file size and modification time
        fcb.size = new_size
        fcb.last_modify = current_time

        # Attempt to update the file content on the disk
        if not self.main_form.disk.file_update(fcb.start, old_size, fcb, content):
            QMessageBox.critical(self, 'Error', 'Failed to save file on disk.')
        else:
            QMessageBox.information(self, 'Success', 'File saved successfully.')
            self.main_form.write_my_disk()
            self.main_form.write_bit_map()
            self.main_form.write_category()
```

#### Creating Directory Tree

```python
    def create_tree(self):
        # Clear existing items in the tree
        self.tree.clear()

        # Define a recursive function to add items
        def add_items(parent_item, node):
            # Create a tree item for the current node
            item = QTreeWidgetItem(parent_item, [node.fcb.file_name])

            # Recursively add tree items for each child node
            for child in node.children:
                add_items(item, child)

        # Check if root node exists
        if self.category.root is not None:
            # Create tree item corresponding to root node
            root_item = QTreeWidgetItem(self.tree, [self.category.root.fcb.file_name + " (Root)"])
            self.tree.addTopLevelItem(root_item)

            # Add tree items for each child of the root node
            for child in self.category.root.children:
                add_items(root_item, child)

            # Expand root node to display all child nodes by default
            root_item.setExpanded(True)
        else:
            print("No root node is defined in the category.")
```

#### Disk File Read and Write Operations

```python
    def read_my_disk(self):
        path = os.path.join(os.getcwd(), "MyDiskInfo.txt")
        if os.path.exists(path):
            with open(path, 'r', encoding='utf-8') as reader:
                # First read disk remaining capacity information
                remain_line = reader.readline().strip()
                if remain_line.startswith("Remaining Blocks:"):
                    self.disk.remain = int(remain_line.split(":")[1].strip())

                for i in range(self.disk.block_num):
                    line = reader.readline()
                    #if line == '\n':  # Check if it's a blank line with only newline character
                        #continue

                    # Decode the line, handling all types of newlines
                    line = line.rstrip("\n")  # Remove only the newline at the end
                    line = line.replace("|||", "\r\n").replace("|r|", "\r").replace("|n|", "\n")

                    self.disk.memory[i] = line

    def write_my_disk(self):
        path = os.path.join(os.getcwd(), "MyDiskInfo.txt")
        if os.path.exists(path):
            os.remove(path)

        with open(path, 'w', encoding='utf-8') as writer:
            # Write disk remaining capacity
            writer.write(f"Remaining Blocks: {self.disk.remain}\n")

            for data in self.disk.memory:
                # Output original data about to be encoded
                print("Original data:", repr(data))

                # Encode the line, handling all types of newlines
                encoded_data = data.replace("\r\n", "|||").replace("\r", "|r|").replace("\n", "|n|")

                # Print encoded data to confirm correct conversion
                print("Encoded data:", repr(encoded_data))

                writer.write(encoded_data + '\n')  # Write converted data plus line separator

```

Special encoding is applied to special characters "\r\n", "\r", "\n". During reading, `line = reader.readline()` is used to ensure reading the entire line including newline characters, then `line = line.rstrip("\n")` removes the trailing newline.

#### Directory File Read and Write Operations

```python
    def read_category(self):
        with open("CategoryInfo.txt", 'r') as file:
            lines = file.readlines()
            root_node = None
            parent_stack = []
            current_node_info = {}

            for line in lines:
                line = line.strip()
                if "Node Start" in line:
                    current_node_info = {}
                elif "Node End" in line:
                    fcb = FCB(current_node_info['File Name'],
                              int(current_node_info['File Type']),
                              current_node_info['Last Modified'],
                              int(current_node_info['File Size']),
                              int(current_node_info['Start Position']))
                    new_node = Category.Node(fcb)
                    if parent_stack:
                        parent_stack[-1].add_child(new_node)
                    else:
                        root_node = new_node  # Mark as root node
                    parent_stack.append(new_node)  # Add current node to stack as parent for subsequent child nodes
                elif "Parent End" in line and parent_stack:
                    parent_stack.pop()  # When all child nodes of a node have been processed, remove that node from stack
                else:
                    if line:
                        parts = line.split(": ", 1)
                        if len(parts) == 2:
                            key, value = parts
                            current_node_info[key.strip()] = value.strip()

        if not root_node:
            default_fcb =  FCB("root", FCB.FOLDER, "", 0)
            root_node = Category.Node(default_fcb)

        self.category.root = root_node
        self.root_node = root_node
        self.current_node = root_node
        self.file_form_init(self.category.root)

    def write_category(self):
        with open("CategoryInfo.txt", 'w') as file:
            def write_node(node, parent_name=""):
                file.write("Node Start\n")
                file.write(f"Parent Name: {parent_name}\n")
                file.write(f"File Name: {node.fcb.file_name}\n")
                file.write(f"File Type: {node.fcb.file_type}\n")
                file.write(f"Last Modified: {node.fcb.last_modify}\n")
                file.write(f"File Size: {node.fcb.size}\n")
                file.write(f"Start Position: {node.fcb.start}\n")
                file.write("Node End\n")
                for child in node.children:
                    write_node(child, node.fcb.file_name)
                file.write("Parent End\n")  # Mark end of parent node

            if self.category.root:
                write_node(self.category.root)

```

The directory structure and file information are written to the directory file. If the directory structure is as follows:

![Directory Structure]({{site.baseurl}}/assets/images/filemanage/directory.png){: .mx-auto .d-block}

Then the content of the directory file will be:

```
Node Start
Parent Name: 
File Name: root
File Type: 1
Last Modified: 2024-04-17 23:25:17
File Size: 0
Start Position: -1
Node End
Node Start
Parent Name: root
File Name: docs
File Type: 1
Last Modified: 2024-04-17 23:24:37
File Size: 0
Start Position: -1
Node End
Parent End
Node Start
Parent Name: root
File Name: res
File Type: 1
Last Modified: 2024-04-17 23:25:04
File Size: 0
Start Position: -1
Node End
Node Start
Parent Name: res
File Name: test
File Type: 1
Last Modified: 2024-04-17 23:24:58
File Size: 0
Start Position: -1
Node End
Parent End
Node Start
Parent Name: res
File Name: project
File Type: 1
Last Modified: 2024-04-17 23:25:04
File Size: 0
Start Position: -1
Node End
Parent End
Parent End
Node Start
Parent Name: root
File Name: yik
File Type: 1
Last Modified: 2024-04-17 23:25:17
File Size: 0
Start Position: -1
Node End
Node Start
Parent Name: yik
File Name: code
File Type: 1
Last Modified: 2024-04-17 23:25:17
File Size: 0
Start Position: -1
Node End
Parent End
Parent End
Parent End
```

To better illustrate the principle, the above file content with indentation added:

```
Node Start
Parent Name: 
File Name: root
File Type: 1
Last Modified: 2024-04-17 23:25:17
File Size: 0
Start Position: -1
Node End

    Node Start
    Parent Name: root
    File Name: docs
    File Type: 1
    Last Modified: 2024-04-17 23:24:37
    File Size: 0
    Start Position: -1
    Node End
    Parent End

    Node Start
    Parent Name: root
    File Name: res
    File Type: 1
    Last Modified: 2024-04-17 23:25:04
    File Size: 0
    Start Position: -1
    Node End

        Node Start
        Parent Name: res
        File Name: test
        File Type: 1
        Last Modified: 2024-04-17 23:24:58
        File Size: 0
        Start Position: -1
        Node End
        Parent End

        Node Start
        Parent Name: res
        File Name: project
        File Type: 1
        Last Modified: 2024-04-17 23:25:04
        File Size: 0
        Start Position: -1
        Node End
        Parent End
    Parent End

    Node Start
    Parent Name: root
    File Name: yik
    File Type: 1
    Last Modified: 2024-04-17 23:25:17
    File Size: 0
    Start Position: -1
    Node End

        Node Start
        Parent Name: yik
        File Name: code
        File Type: 1
        Last Modified: 2024-04-17 23:25:17
        File Size: 0
        Start Position: -1
        Node End
        Parent End
    Parent End
Parent End

```

The file exhibits the following characteristics:

* Each node starts with a "Node Start" marker and ends with a "Node End" marker;
* When all child nodes of a node have been listed, "Parent End" is output;

These characteristics can also be obtained through the `write_category(self)` function;

Next, the principle of the `read_category(self)` function is explained:

* First, all data from the directory file is read;
* Process the read data line by line:
  * If the line contains "Node Start", clear current_node_info to prepare for reading new node data;
  * If it's other node information, split the data to obtain key and value;
  * If the line contains "Node End", it indicates all information for a node has been read. Create the node and add it as a child node to the current parent node (top element of stack). If the stack is empty at this point, this node is the root node. Finally, push this node onto the stack to serve as the parent for subsequent nodes;
  * If the line contains "Parent End", it indicates all child nodes of the current parent have been processed, so pop the top element from the stack;
* If there is no root node information, create a default root node;
* Set the root node of the directory;