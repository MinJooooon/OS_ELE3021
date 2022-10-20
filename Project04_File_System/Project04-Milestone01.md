# Objective
+ Expand the maximum size of a file by implementing a triple indirect block

# Overall Overview
![indirect](uploads/13ae16fb3197414fd0266d50f6557c0a/indirect.png)
+ One inode has 13 pointers
  + 10 Direct pointers
  + 1 Single indirect pointer
  + 1 Doubly indirect pointer
  + 1 Triple indirect pointer

# Defined constants
```c
#define NDIRECT 10
#define NINDIRECT (BSIZE / sizeof(uint))

#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define NTINDIRECT (NINDIRECT * NINDIRECT * NINDIRECT)

#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT + NTINDIRECT)

// On-disk inode structure
struct dinode {
  ...

  ...
  uint addrs[NDIRECT+3];   // Data block addresses
};
```
+ `fs.h`
  + `NDIRECT`: changed from 12 to 10 to follow the instruction of project
  + `NINDIRECT`: the same as original xv6(actual value: 128)
  + `NDINDIRECT`: the size of doubly indirect block(actual value: 128 * 128)
  + `NTINDIRECT`: the size of triple indirect block(actual value: 128 * 128 * 128)
  + `MAXFILE`: changed from (NDIRECT + INDIRECT) to (NDIRECT + NINDIRECT + NDINDIRECT + NTINDIRECT)
  + `addrs`: the value of NDIRECT was changed, so reflect the changed size from `NDIRECT + 1` to `NDIRECT + 3`


```c
struct inode {
  ...

  ...
  uint addrs[NDIRECT+3];
};
```
+ `file.h`
  + `addrs`: the value of NDIRECT was changed, so reflect the size value from `NDIRECT + 1` to `NDIRECT + 3`

```c
...
#define FSSIZE 40000 // size of file system in blocks
```
+ `param.h`
  + `FSSIZE`: to accommodate the bigger size of file, changed from 1000 to 40000

# `static uint bmap(struct inode *ip, uint bn)` in `fs.c`

```c
uint addr, *a;
struct buf *bp;

if(bn < NDIRECT){
  if((addr = ip->addrs[bn]) == 0)
    ip->addrs[bn] = addr = balloc(ip->dev);
  return addr;
}
```
+ The above code is for direct pointer
+ It was originally implemented in xv6
+ Find the block with the given parameter `bn`
+ If it doesn't exist, allocate new block by using `balloc()`

```c
bn -= NDIRECT;

if(bn < NINDIRECT){
  // Load indirect block, allocating if necessary.
  if((addr = ip->addrs[NDIRECT]) == 0)
    ip->addrs[NDIRECT] = addr = balloc(ip->dev);
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[bn]) == 0){
    a[bn] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);
  return addr;
}
```
+ The above code is for single indirect pointer(`ip->addrs[NDIRECT]`)
+ It was originally implemented in xv6
+ If block number exceeds the space for direct pointer, find the block of indirect pointer
+ If it doesn't exist, allocate new block by using `balloc()`

```c
bn -= NINDIRECT;

if(bn < NDINDIRECT) {
  if((addr = ip->addrs[NDIRECT + 1]) == 0) {
    ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
  }

  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[bn / NINDIRECT]) == 0) {
    a[bn / NINDIRECT] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);
    
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[bn % NINDIRECT]) == 0) {
    a[bn % NINDIRECT] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);

  return addr;
}
```
+ The above code is for doubly indirect pointer(`ip->addrs[NDIRECT + 1]`)
+ First, calculate the index of block pointed by doubly indirect pointer
  + bn / NINDIRECT
+ Second, calculate the index of address block pointed by the block calculated above
  + bn % NINDIRECT

```c
bn -= NDINDIRECT;

if(bn < NTINDIRECT) {
  if((addr = ip->addrs[NDIRECT + 2]) == 0) {
    ip->addrs[NDIRECT + 2] = addr = balloc(ip->dev);
  }
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[bn / NDINDIRECT]) == 0) {
    a[bn / NDINDIRECT] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);

  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[(bn % NDINDIRECT) / NINDIRECT]) == 0) {
    a[(bn % NDINDIRECT) / NINDIRECT] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);

  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[(bn % NDINDIRECT) % NINDIRECT]) == 0) {
    a[(bn % NDINDIRECT) % NINDIRECT] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);

  return addr;
}
```
+ The above code is for triple indirect pointer(`ip->addrs[NDIRECT + 2]`)
+ First, calculate the index of depth-1 block pointed by triple indirect pointer
  + bn / NDINDIRECT
+ Second, calculate the index of depth-2 block pointed by depth-1 block calculated above
  + (bn % NDINDIRECT) / NDINDIRECT
+ Finally, calculate the index of address block pointed by depth-2 block calculated above
  + (bn % NDINDIRECT) % NINDIRECT

# `static void itrunc(struct inode *ip)` in `fs.c`

```c
int i, j, k, l;
struct buf *bp, *dbp, *tbp;
uint *a, *da, *ta;

for(i = 0; i < NDIRECT; i++){
  if(ip->addrs[i]){
    bfree(ip->dev, ip->addrs[i]);
    ip->addrs[i] = 0;
  }
}
```
+ The above code is for direct pointer(`i = 0; i < NDIRECT; i++`)
+ It was originally implemented in xv6
+ Free the blocks for direct pointer

```c
if(ip->addrs[NDIRECT]){
  bp = bread(ip->dev, ip->addrs[NDIRECT]);
  a = (uint*)bp->data;
  for(j = 0; j < NINDIRECT; j++){
    if(a[j])
      bfree(ip->dev, a[j]);
  }
  brelse(bp);
  bfree(ip->dev, ip->addrs[NDIRECT]);
  ip->addrs[NDIRECT] = 0;
}
```
+ The above code is for single indirect pointer(`NDIRECT`)
+ It was originally implemented in xv6
+ Free the blocks
  + block pointed by single indirect pointer
  + indirect pointer

```c
if(ip->addrs[NDIRECT + 1]) {
  bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
  a = (uint*)bp->data;
  for(j = 0; j < NINDIRECT; j++) {
    if(a[j]) {
      dbp = bread(ip->dev, a[j]);
      da = (uint*)dbp->data;
      for(k = 0; k < NINDIRECT; k++) {
        if(da[k]) {
          bfree(ip->dev, da[k]);
        }
      }
      brelse(dbp);
      bfree(ip->dev, a[j]);
    }
  }
  brelse(bp);
  bfree(ip->dev, ip->addrs[NDIRECT + 1]);
  ip->addrs[NDIRECT + 1] = 0;
}
```
+ The above code is for doubly indirect pointer(`NDIRECT + 1`)
+ Free the blocks
  + block pointed by intermediate block
  + block pointed by doubly indirect pointer
  + doubly indirect pointer

```c
if(ip->addrs[NDIRECT + 2]) {
  bp = bread(ip->dev, ip->addrs[NDIRECT + 2]);
  a = (uint*)bp->data;
  for(j = 0; j < NINDIRECT; j++) {
    if(a[j]) {
      dbp = bread(ip->dev, a[j]);
      da = (uint*)dbp->data;
      for(k = 0; k < NINDIRECT; k++) {
        if(da[k]) {
          tbp = bread(ip->dev, da[k]);
          ta = (uint*)tbp->data;
          for(l = 0; l < NINDIRECT; l++) {
            if(ta[l]) {
              bfree(ip->dev, ta[l]);
            }
          }
          brelse(tbp);
          bfree(ip->dev, da[k]);
        }
      }
      brelse(dbp);
      bfree(ip->dev, a[j]);
    }
  }
  brelse(bp);
  bfree(ip->dev, ip->addrs[NDIRECT + 2]);
  ip->addrs[NDIRECT + 2] = 0;
}
```
+ The above code is for triple indirect pointer(`NDIRECT + 2`)
+ Free the blocks
  + block pointed by depth-2 block
  + depth-2 block
  + depth-1 block
  + triple indirect pointer