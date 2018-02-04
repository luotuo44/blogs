##多个目录多个makefile文件

假定有以下的文件结构
```
project
|	main.cpp
|	makefile
|
|___sub
|	|	sub.h
|	|	sub.cpp
|	|	makefile
|
|
|___add
 	|	add.h
 	|	add.cpp
 	|	makefile

```

显然，我们想要的效果是：在`project`目录中，输入`make`和`make clean`能做到编译/清理子目录的内容。

**`sub`目录的`makefile`文件如下**

```

TARGET = libsub.a
BUILD_DIR = build

FLAGS = -Wall -Werror -std=c++11
src = $(wildcard *.cpp)
objects = $(patsubst %.cpp, $(BUILD_DIR)/%.o, $(src))


.PHONY:all
all: PRE $(TARGET)
	@echo "finish compiling sub module"


#.PHONY:PRE
PRE:
	mkdir -p $(BUILD_DIR)


$(TARGET):$(objects)
	ar -cq $@ $^


$(BUILD_DIR)/%.o : %.cpp 
	g++ -c $(FLAGS)  $<  -o $@


.PHONY:clean
clean:
	rm -rf build/*
	rm -rf $(TARGET)

```



**`add`目录的`makefile`目录如下：**

```

TARGET = libadd.a
BUILD_DIR = build

FLAGS = -Wall -Werror -std=c++11
src = $(wildcard *.cpp)
objects = $(patsubst %.cpp, $(BUILD_DIR)/%.o, $(src))


.PHONY:all
all: PRE $(TARGET)
	@echo "finish compiling add module"



.PHONY:PRE
PRE:
	mkdir -p $(BUILD_DIR)

$(TARGET):$(objects)
	ar -cq $@ $^


$(BUILD_DIR)/%.o : %.cpp 
	g++ -c $(FLAGS)  $<  -o $@


.PHONY:clean
clean:
	rm -rf build/*
	rm -rf $(TARGET)

```


**`project`目录的`makefile`文件如下：**

```

FLAGS = -Wall -Werror -std=c++11

TARGET = main

BUILD_DIR = build


LIBDIR = -Ladd -Lsub
LIBS = -ladd -lsub

SUB_DIR = add sub

src = $(wildcard *.cpp)
objects = $(patsubst %.cpp, $(BUILD_DIR)/%.o, $(src))


.PHONY:all
all: PRE $(TARGET)
	@echo "all compiling finsih"


.PHONY:PRE
PRE:
	mkdir -p $(BUILD_DIR)


$(TARGET):$(objects) $(SUB_DIR)
	g++ $(objects) $(FLAGS) $(LIBDIR) $(LIBS) -o $(TARGET)


$(BUILD_DIR)/%.o: %.cpp
	g++ -c $(FLAGS) $< -o $@


.PHONY:$(SUB_DIR)
$(SUB_DIR):
	for dd in $(SUB_DIR); do \
		make -C $$dd; \
	done



.PHONY:clean
clean: clean_sub_dir
	rm -rf $(BUILD_DIR)/*.o
	rm -rf $(TARGET)


.PHONY:clean_sub_dir
clean_sub_dir:
	for dd in $(SUB_DIR); do \
		make clean -C $$dd; \
	done

```


每个`makefile`都会在当前目录建立一个`build`目录，然后把目标文件写入到该文件中。
**需要注意的是：** `main.cpp`文件内需要带路径地包含子目录的头文件，比如`#include"sub/sub.h"`。


`PS`:
```
$@:表示目标文件
$^:表示所有依赖文件
$<:表示第一个依赖文件
```

wildcard是扩展通配符函数，其作用是展开成一列所有符合由其参数描述的文件名，文件名以空格间隔。一般用于列出特定目录下的某类文件。列出sub目录的所有`cpp`文件写法如下：
```
srcs = $(wildcard sub/*.cpp)
```

notdir函数则是去掉文件名中的路径信息，用法如下：
```
files =$(notdir $(srcs))
```

patsubst为匹配替换函数， patsubst(需要匹配的样式，替换的目标，需要匹配的字符串)函数。一般用于根据源文件名得到相应的目标文件名。
```
$(patsubst %.cpp, %.o, $(srcs))
```
