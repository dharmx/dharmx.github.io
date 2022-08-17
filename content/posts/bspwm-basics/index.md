---
draft: true

title: "Bspwm Basics"
summary: "This is a record of me trying to understand how BSPWM works
          by analysing the source code and whatnot."
description: "Understanding the working and intricacies of BSPWM."

date: 2022-08-24T17:29:27+05:30
lastmod: 2022-08-24T17:29:27+05:30

author: "dharmx"
authorLink: "https://dharmx.is-a.dev"

tags: ["bspwm", "wm"]
keywords: ["wm", "linux", "desktop", "x11"]
categories: ["deepdive"]
relative: true

resources:
- name: "featured-image"
  src: "images/featured-image.png"

images: ["/bspwm-basics/images/featured-image.png"]
toc:
  auto: true

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: false
ruby: false
fraction: false
fontawesome: false
linkToMarkdown: true
rssFullText: false
---

Automation and workflow feat. BSPWM.

# Introduction

BSPWM is a neat window manager with some neat features. In this article
we will be briefly looking at how BSPWM functions and what some of its features
are.

Maybe analyze a bit of its source code, IPC commands and some other possible
extensions.

## Before We Begin

This is a fairly intermediate article which may seem easy to follow at first but,
will get relatively difficult to follow as it progresses further. You would need
to have the following things to progress smoothly.

- Maybe a tiny amount of BST data structure.
- Experience with window managers and X11.
- Programming experience is preferred.

All right! Let us begin.

## About BSPWM

BSPWM is a tiling window manager that represents windows as the leaves of a **full**
binary tree. It uses an IPC application that invokes X events and changes BSPWM'
internal states.

Unlike most other window managers it does not have a dedicated configuration file.
You use the IPC client (bspc) for everything. Be it setting window gaps, border
colors, layouts, monitors, etc.

### The BST data structure.

The Binary Tree data structure is a way of representing data in the form of a
tree. A BST is a special case of tree that only has two children.
As all trees (both natural and pragmatic) this one also has leaves and branches.
The leaves are called nodes which is the term we will use from now on.

{{< figure
  src="./svgs/bst.svg"
  title="A Binary Tree"
  caption="A brief descriptive illustration of a binary tree."
  align="center"
>}}

You may also notice that the nodes are colored differently.
The colors signify levels. For instance, the root level node is colored
with {{< mark class="bspwm-basics-mark-index-green" >}}green{{< /mark >}}. The second
level node is colored with {{< mark class="bspwm-basics-mark-index-blue" >}}blue{{< /mark >}}
and lastly the third level node is colored with
{{< mark class="bspwm-basics-mark-index-red" >}}red{{< /mark >}}.

A BST node would look like the following in C.

```c
struct node_t {
    int some_data;
    node_t *left_child;
    node_t *right_child;
};
```

This node will contain a pointer to the left child and another to the right child.
Left and right child are siblings as illustrated in the diagram. And,
`some_data` which is the data that the node will be carrying.
Now, consider the following code to get a clearer visualization of how the node
is being initialized and how some basic components work.

{{< collapse summary="Minimal BST implementation in C" >}}

```c
typedef struct node_t node_t;
struct node_t {
    int data;
    node_t* left;
    node_t* right;
};

int
main(void) {
    node_t root;
    node_t left;
    node_t left_left;
    node_t left_right;
    node_t right;

    root.data       = 10;
    left.data       = 11;
    left_left.data  = 124;
    left_right.data = 89;
    right.data      = 19;

    root.right = &right;
    root.left  = &left;
    left.left  = &left_left;
    left.right = &left_right;

    printf("   %d\n"
           "    |\n"
           "   / \\\n"
           "  %d  %d\n"
           "   |\n"
           "  / \\\n"
           "%d  %d\n",
           root.data,
           root.left->data,
           root.right->data,
           root.left->left->data,
           root.left->right->data);
    return 0;
}

// vim:filetype=c
```

```
   10
    |
   / \
  11  19
   |
  / \
124  89
```

{{< /collapse >}}

The above code is a recreation of the binary tree diagram that you came
across in the beginning. Play with it to get a feel for BSTs.

Similarly, BSPWM interprets a window as a node and then maps them into a tree.
Following is a part of BSPWM source code that defines a tree node.
Seems quite intuitive and easy to understand right?

{{< admonition >}}
Do not try to understand what each variable does. Just take a note
of the similarities between the two node definitions.
{{< /admonition >}}

```c
typedef struct node_t node_t;
struct node_t {
    uint32_t id;
    split_type_t split_type;
    double split_ratio;
    presel_t *presel;
    xcb_rectangle_t rectangle;
    constraints_t constraints;
    bool vacant;
    bool hidden;
    bool sticky;
    bool private;
    bool locked;
    bool marked;
    node_t *first_child;
    node_t *second_child;
    node_t *parent;
    client_t *client;
};
```

### Display Structure

It should be noted that BSPWM is a window manager, which means it manages
windows. And it does so by representing the window tree as leaves of a
full BST.

For application windows to be able to spawn BSPWM first needs to define a
root window. Which in WM lingo is called a workspace. And, after said
workspace(s) are defined one can now finally open an application window
which will be managed by BSPWM.

Add on top of this information it is to be noted that BSPWM was originally
designed for a computer with only one monitor. If you are curious then
yes, the author has already added multi-monitor support. Monitors are also
treated the same as roots i.e. a list.

If you are still confused then the following illustration will help ease
that.

{{< figure
  src="./svgs/bspwm-mon-ws.svg"
  title="BSPWM monitor-workspace relation"
  caption="How BSPWM defines the relation between monitors and workspaces."
  align="center"
>}}

This is the easiest part because it is intuitive. According to the figure
there are two monitors which can be represented with an array. And workspaces
which are branching out of the monitor.

### Monitors

If we analyze the source then we will see that workspaces are not represented
as a multi-branched tree rather, a single branched one i.e. a linked list.
Where the current desktop has links to the previous and next desktops.

{{< highlight c "anchorlinenos=true,hl_inline=false,linenos=inline,hl_lines=303 16-17,linenostart=288" >}}
typedef struct monitor_t monitor_t;
struct monitor_t {
    char name[SMALEN];
    uint32_t id;
    xcb_randr_output_t randr_id;
    xcb_window_t root;
    bool wired;
    padding_t padding;
    unsigned int sticky_count;
    int window_gap;
    unsigned int border_width;
    xcb_rectangle_t rectangle;
    desktop_t *desk;
    desktop_t *desk_head;
    desktop_t *desk_tail;
    monitor_t *prev;
    monitor_t *next;
};
{{< /highlight >}}

```c
// code representation.
const int MON_LEN = 2;
monitor_t monitors[MON_LEN];
```

See those `prev` and `next` variables? Intuitively, the `prev` variable points
to the previous `monitor_t` structure and `next` next. And, if either of those
pointer variables have `NULL` then it is to be assumed that the `focused`
monitor is at the end or, the beginning of the monitor list.

### Workspaces

For desktops, the structure is pretty much the same as monitors.
That is, both desktops and monitors are represented as doubly linked lists
with links to previous and next desktops.

Therefore, see the following code snippet.

{{< highlight c "anchorlinenos=true,linenos=inline,hl_lines=281 9-10,linenostart=273" >}}
typedef struct desktop_t desktop_t;
struct desktop_t {
    char name[SMALEN];
    uint32_t id;
    layout_t layout;
    layout_t user_layout;
    node_t *root;
    node_t *focus;
    desktop_t *prev;
    desktop_t *next;
    padding_t padding;
    int window_gap;
    unsigned int border_width;
};
{{< /highlight >}}

### Monitor-Workspace Relation

The following figure attempts to briefly define the monitor-workspace
relation in BSPWM that takes the above `monitor_t` and `desktop_t`
definitions into account.

{{< figure
  src="./svgs/linked-list-bspwm.svg"
  title="BSPWM Monitor-Workspace Relation"
  caption="Representing monitor-workspace in the form of linked lists."
  align="center"
>}}

The first monitor (marked in green) has three desktops where the label
of the first desktop is identified by `(+)`, second by `(-)` and lastly
third by `(=)`.

It should be worth noting that having duplicate labels are possible
because the window manager does not identify workspaces from labels
rather they use the ID of that desktop/workspace. You can check what
the current IDs with a simple query to BSPWM via the `bspc` IPC client.

```sh
$ bspc query --desktops # IDs
0x00400004
0x00400005
0x00400006

$ bspc query --names --desktops # Labels
(+)
(-)
(=)
```

### Windows

This is the actual important part. This part makes BSPWM stand out from all
other window managers. The previous parts are generally the same for all
window managers.
