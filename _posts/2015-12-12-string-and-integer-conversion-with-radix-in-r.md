---
layout: post
title: String与Integer的含radix转换(R)
category: tech
guid: C2383C86-6048-4EFE-9CE5-0E14B2BC3EB9
tags: [R]

---
{% include JB/setup %}

在实际应用中，有时需要生成唯一标识符, 如id。一种常见的生成方式是利用系统
当前的UNIX TIMESTAMP加上流水号（如在并行环境中）, 毕竟时间是单向流动的嘛(?)。
但是系统的当前时间值毕竟还是很大，直接将其作为id有些太长，有时希望将其转换成长度短一点的字符串。

在`R`中将String转成Integer的函数是`base::strtoi(str, radix)`, 但是没有找到对应的`itostr(num, radix)`。
系统中倒有一个函数`as.character`， 但是这是以10为基的转换。下文是整理的String与Integer之间含radix的转换函数,
radix为`[1-36]`(`R`中可用`as.numeric(Sys.time())`获取系统当前的UNIX TIMESTAMP):

```R
# string and integer conversion with radix
# MAX_RADIX = 36

itostr <- function(num, radix) {

  # 1. check input
  MAX_RADIX <- 36
  MAX_RADIX_STR <- tolower(c(as.character(0:9), letters))

  num <- gsub(pattern = "\\s", replacement = "", x = paste(num, collapse = "|"), perl = T)
  if (!grepl(pattern = "^\\d+$", x = num, perl = T)) {
    stop(paste("num: ", paste(num, collapse = "|"), "不是正整数", sep = ""))
  } else {
    num <- as.numeric(num)
  }
  
  radix <- gsub(pattern = "\\s", replacement = "", x = paste(radix, collapse = "|"), perl = T)
  if (!grepl(pattern = "^(([1-9])|([1-2][1-9])|(3[1-6]))$", x = radix, perl = T)) {
    stop(paste("radix:", paste(radix, collapse = "|",  "不是[1-", MAX_RADIX, "]的正整数", sep = "")))
  } else {
    radix <- as.numeric(radix)
  }
  
  # 2. processing
  ret <- ""
  while (num >= radix) {
    idx <- num%%radix + 1
    char <- MAX_RADIX_STR[idx]
    
    ret <- paste(char, ret, sep="")
    num <- floor(num/radix)
  }
  
  first_char <- MAX_RADIX_STR[num%%radix + 1]
  ret <- paste(first_char, ret, sep="")
  
  ret
}


# string to integer with radix
strtoi <- function(str, radix) {
  
  # 1. check input 
  MAX_RADIX <- 36
  MAX_RADIX_STR <- tolower(c(as.character(0:9), letters))

  radix <- gsub(pattern = "\\s", replacement = "", x = paste(radix, collapse = "|"), perl = T)
  if (!grepl(pattern = "^(([1-9])|([1-2][0-9])|(3[1-6]))$", x = radix, perl = T)) {
    stop(paste("radix:", paste(radix, collapse = "|",  "不是[1-", MAX_RADIX, "]的正整数", sep = "")))
  } else {
    radix <- as.numeric(radix)
  }
  
  
  str <- gsub(pattern = "\\s", replacement = "", x = paste(str, collapse = "|"), perl = T)
  if (!grepl(pattern = "^([0-9a-z])+$", x = str, ignore.case = T, perl = T)) {
    stop(paste(paste(str, collapse = "|"), 
               "含有非[", 
               paste(radix_str[1], "-", radix_str[length(radix_str)], sep = ""),
               "]的字符", 
               sep = ""))
  }
  
  # 2. processing
  str_reshape <- as.character(str)
  str_vec <- tolower(strsplit(str_reshape, "")[[1]])
  len <- length(str_vec)
  
  ret <- 0
  for (k in len:1) {
    t_idx <- match(str_vec[k], MAX_RADIX_STR)-1 # 由于radix_str中0的存在
    ret <- ret + t_idx * radix^(len-k)
  }
  
  ret
}
```

#### 改进方向
1. 支持`vector`参数


*The End.*
