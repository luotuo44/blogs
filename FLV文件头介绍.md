FLV文件介绍
========

总体结构
=======
FLV文件分为两个部分:文件头和数据部分。文件头包含的基本信息(即描述视频文件的元数据)并不多。数据部分是由一个个的tag组成，有ScriptData、Video和Audio这三种tag。其中ScriptData tag包含了视频本身的很多元数据。Video tag和Audio tag则包含对应的音视频内容。一个flv文件一般只包含一个ScriptData tag，以及多个Video和Audio tag。结构如下:

|文件头|
| :---: |
|ScriptData Tag |
|Video Tag      |
|Video Tag		|
|Aduio Tag 		|
|Aduio Tag 		|
|Aduio Tag 		|


盗用[雷神](http://blog.csdn.net/leixiaohua1020/article/details/17934487)的一张图，FLV的文件头和Tag简单如下：
![flv](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/flv.jpg)

ScirptData、Video和Audio这三种tag具有相同的Tag Header，通过第一个字节(Type)标明具体是何种tag。不同的是Tag Data字段。

# 文件头
参考前面的图示，FLV的文件头可以一目了然。


# ScriptData Tag
Script Tag一般是第一个tag，用于表示该FLV文件的元数据，因此FLV文件只有一个ScriptData Tag。

Script Tag采样[AMF](https://en.wikipedia.org/wiki/Action_Message_Format#AMF0)二进制方式序列化数据。FLV的ScriptData Tag有字符、浮点数、字符串、日期、数组和对象这几种类型。*序列化时，每种类型前面都用一个字节标识紧接着的是何种类型。因此，反序列化时先读取标识符，接着根据标识符指明的类型解读紧接着的二进制*。显然，在反序列化时，需要建立一个映射关系，类型和相应的处理函数的映射关系，因为不同的类型有着不同的读取方式。
标识符和对应的类型如下表所示。**[Adobe Flash Video File Format Specification Version 10.1](https://download.macromedia.com/f4v/video_file_format_spec_v10_1.pdf)文档将下表中的类型统称为`SCRIPTDATAVALUE`**。

|标识符|对应的类型|说明|
|:---:|:------:|:---:|
|0|Number|数值(包括整数和浮点数)，采用double存储|
|1|Boolean|0x00或0x01,采样char存储|
|2|SCRIPTDATASTRING|字符串|
|3|SCRIPTDATAOBJECT|对象|
|4|MovieClip|保留字段|
|5|Null|-|
|6|Undefined|-|
|7|UI16|Reference(两个字节)|
|8|SCRIPTDATAECMAARRAY|key-value数组|
|9|SCRIPTDATAOBJECTEND|SCRIPTDATAOBJECT的结束符|
|10|SCRIPTDATASTRICTARRAY|SCRIPTDATAVALUE数组|
|11|SCRIPTDATADATE|日期(unix时间戳和timezone offset)|
|12|SCRIPTDATALONGSTRING|长字符串|


接下来先易后难地解释各个类型，并附上简单的读取代码。
## 简单类型
### Number
Number字段的内容和C/C++中的double一致，直接读取8字节即可。唯一值得注意的是大小端字节序问题。读取代码如下：

```cpp
inline bool isLittleEndian()
{
    int a = 1;
    return *(reinterpret_cast<char*>(&a));
}

double parseMetadataDouble(std::ifstream &in)
{
	static_assert(sizeof(double) == 8, "sizeof(double) not equals to 8");
	char type;
	in.read(&type, 1);
	assert(type == 0);//为Double类型

    double d = 0;
    in.read(reinterpret_cast<char*>(&d), 8);
    if(isLittleEndian())
        std::reverse(reinterpret_cast<char*>(&d), reinterpret_cast<char*>(&d)+8);

	std::cout<<d<<std::endl;//最终的值
	return d;
}

```

### String
string(2)和long string(12)这两种字符串的value读取方式一致，前面2个(string)或4个字节(long string)表示长度(假设为N)，紧接着的N个字节为实际的字符串内容。[Adobe FLV Format](https://download.macromedia.com/f4v/video_file_format_spec_v10_1.pdf)的说明如下：
![string_comment](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/string.jpg)

![long_string_comment](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/long_string.jpg)

读取代码如下：
```cpp

using uchar = unsigned char;

template<typename InputIterator>
inline size_t getSizeValue(InputIterator begin, InputIterator end)
{
    size_t init = 0;
    return std::accumulate(begin, end, init, [](size_t val, uchar cur){
        return (val<<8) + cur;
    });
}


std::string parseMetadataString(std::ifstream &in)
{
	char type;
	in.read(&type, 1);
	assert(type == 2 || type == 12);//为字符串类型

	
    std::vector<uchar> str_len_vec(type & 0x6);
    in.read(reinterpret_cast<char*>(str_len_vec.data()), str_len_vec.size());
    size_t str_len = getSizeValue(str_len_vec.begin(), str_len_vec.end());

	std::string str;
    if(str_len > 0)
    {
        str.assign(str_len, '\0');
        in.read(&str[0], str_len);
        std::cout<<str;
    }

	return str;
}

```

### Date
日期是通过Unix时间戳表示，单位为毫秒。因此不能通过32位的整型表示，其是通过double存储的。[Adobe FLV Format](https://download.macromedia.com/f4v/video_file_format_spec_v10_1.pdf)的说明如下：
![date](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/date.jpg)

读取代码如下：
```cpp
inline bool isLittleEndian()
{
    int a = 1;
    return *(reinterpret_cast<char*>(&a));
}


void parseMetadataDate(std::ifstream &in)
{
	char type;
	in.read(&type, 1);
	assert(type == 11);//为Date 类型

	
    double d = 0;
    in.read(reinterpret_cast<char*>(&d), sizeof(d));
	
	if(isLittleEndian())
    	std::reverse(reinterpret_cast<char*>(&d), reinterpret_cast<char*>(&d)+8);

    time_t ts = static_cast<time_t>(d/1000);//d是毫秒
    std::string str_ts(ctime(&ts));
    std::cout<<str_ts;

	int16_t time_offset;
	in.read(&time_offset, 2);//虽然timezone offset用不着，但仍然需要读出来。
}
```

## 复杂类型
介绍完简单的类型，接着介绍复杂的类型。复杂是因为本身就是一个组合类型。
留意`strict array`和`ecma array`这两个数组，其定义分别如下：
![strict_array](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/strict_array.jpg)

![ecma_array](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/ecma_array.jpg)

`strict array`本身为一个`SCRIPTDATAVALUE`数组，明显这需要递归解读AMF格式。`ecma array`看起来友善一些，是一个object_property数组，但实际并非如此，且看`SCRIPTDATAOBJECTPROPERTY`的定义：

![object_property](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/object_property.jpg)

显然，`ecma array`也包含了一个`SCRIPTDATAVALUE`数组,所不同的是`strict array`是一个单纯的`SCRIPTDATAVALUE`数组, 而`ecma array`则是一个key-value数组，key为`String`类型， value为`SCRIPTDATAVALUE`类型。


除了`strict array`和`ecma string`这两种复杂类型外，`SCRIPTDATAVALUE`还包括`SCRIPTDATAOBJECT`类型，其定义如下:
![object](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/object_and_object_end.jpg)

object类型是由一个`object property`数组和结束符组成，存在结束符字段是因为其没有数组长度字段，这使得读取时需要做一些特别的处理。结合`object property`、`object end`和String类型的特点，可以先读取两个字节，如果两个字节有一个非0，那么为字符串String的长度值；如果皆为0，则说明是`object end`的前面两个字节。读取代码如下：
```cpp
bool parseObjectProperty(std::ifstream &in)
{
    std::string property_name = parseMetadataString(in);//读取一个short string
    //ojbect是由ObjectProperty数组组成，但没有长度信息，只有一个结束符信息。因此需要判断是否为结束符
    //结束符用00 00 09三个字节表示，结合ObjectProperty的前两个字节表示string长度，可以得出：
    //如果前两个字节为00 00，那么就可以确定到达了结束符，最后一个字节留给父项读取
    if( property_name.empty() )
        return false;

    std::cout<<property_name<<" = ";

    parseDataValue(in);//读取DataValue

    return true;
}


void parseObject(std::ifstream &in)
{
    while(parseObjectProperty(in))
    {
        //do nothing
    }

	char end = '\0';
	in.read(&end, 1);
    assert(end == 9);//结束符, 前两个字节在parseObjectProperty==>readString中已经被读取
}
```


现在回过头看一下`strict array`和`ecma array`的读取方法。
```cpp

size_t readInt(std::ifstream &in, size_t bytes)
{
    if( bytes == 0)
        return 0;

    std::vector<uchar> vec(bytes);
	in.read(reinterpret_cast<char*>(vec.data()), bytes);
    size_t val = getSizeValue(vec.begin(), vec.end());

    return val;
}


void parseEcmaArray(std::ifstream &in)
{
    size_t array_len = readInt(in, 4);
    std::cout<<"EcmaArray len = "<<array_len<<std::endl;
    for(size_t i = 0; i < array_len; i += 1)
    {
        std::cout<<"EcmaArray index = "<<i<<"\t";
        parseObjectProperty(in);
        std::cout<<std::endl;
    }


    size_t end_token = readInt(in, 3);
    assert(end_token == 9);
    (void)end_token;
}


void parseStrictArray(std::ifstream &in)
{
    size_t array_len = readInt(in, 4);
    for(size_t i = 0; i < array_len; i += 1)
    {
        std::cout<<" StrictArray index = "<<i<<"\t";
        parseDataValue(in);
    }
}
```
上面的代码中，就差一个parseDataValue没有实现。我们的最初的目的就是解析ScriptData Tag，而ScriptData Tag的数据本身就是一个DataValue。在一开始，就提及需要建立类型和相应的函数处理映射关系。因此parseDataValue()函数的内部实现也是比较简单的，就是读取一个字节，判断其类型，接着调用相应的处理函数(也就是前面介绍的各个类型读取函数)。

```cpp
void parseDataValue(std::ifstream &in)
{
    char type;
	in.read(&type, 1);

    std::cout<<"(type: "<<static_cast<int>(type)<<") ";
    if(op.find(type) != op.end())
        op[type](in);
    else
    {
        std::cout<<"unexpection type "<<static_cast<int>(type)<<std::endl;
        exit(1);
    }
}
```
代码中的`op`可以是一个全局变量，类型为`std::map<int, std::function<void (std::ifstream &)>`。初始化`op`是体力活，这里不介绍了。可以直接观看[源码](https://github.com/luotuo44/FlvExploere/blob/master/ScriptDataTag.cpp#L37)。


## 元数据内容
一般来说ScriptData Tag包含两个AMF包，即两个`SCRIPTDATAVALUE`。并且第一个为字符串类型，其值为`onMetaData`；第二个则为`ecma array`类型，数组包括许多关于FLV文件的基本信息，比如：视频的分辨率、时长、帧率、采样率、音视频的编码器等等。典型的有下面几种:

|属性|类型|说明|
|:--:|:--:|:--:|
|audiocodecid| Number|音频编码器 |
|audiodatarate| Number| 音频流码率(单位为Kb/s)|
|audiodelay| Number| Delay introduced by the audio codec in seconds|
|audiosamplerate| Number| 音频采样率|
|audiosamplesize| Number| Resolution of a single audio sample|
|canSeekToEnd| Boolean| Indicating the last video frame is a key frame
|creationdate| String| 创建时间|
|duration| Number| FLV时长(单位为秒)|
|filesize| Number| 文件尺寸(单位为字节)|
|framerate| Number| 视频帧率|
|height| Number| 视频高度|
|stereo| Boolean| Indicating stereo audio|
|videocodecid| Number|视频编码器 | 
|videodatarate| Number| 视频流码率(单位为Kb/s)|
|width| Number| 视频宽度|
|hasKeyframes| Boolean | 是否有关键帧|
|hasVideo|Boolean|是否有视频流|
|hasAudio|Boolean|是否有音频流|
|keyframes|SCRIPTDATAOBJECT|关键帧信息|


### keyframes
元数据keyframes是一个重要的元数据(但并非官方标准)，它记录视频流的关键帧在文件中的位置以及对应的时间。Object类型本身一个是`object property`数组，在此，它是由两个key-value组成的数组。两个key都是字符串，分别为`filepositions`和`times`。而value则是一个`strict array`类型，分别由所有关键帧的文件位置和时间信息组成。下图的读取的一个例子:
![key_frames](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/keyframes_example.jpg)


# VieoData Tag
继续盗用[雷神](http://blog.csdn.net/leixiaohua1020/article/details/17934487)的图，video tag数据部分的第一个字节注明视频数据的帧类型已经编码类型，如下：
![video_data_tag](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/video_data_tag.jpg)

其中，帧类型有下面几种：

|值|类型|
|:--:|:--:|
|1|key frame (for AVC, a seekable frame)|
|2|inter frame (for AVC, a non-seekable frame)|
|3|disposable inter frame (H.263 only)|
|4|generated key frame (reserved for server use only)|
|5|video info/command frame|

视频编码类型有下面几种：

|值|类型|
|:--:|:--:|
|2|Sorenson H.263|
|3|Screen video|
|4|On2 VP6|
|5|On2 VP6 with alpha channel|
|6|Screen video version 2|
|7|AVC|



# AudioData Tag
继续上[雷神](http://blog.csdn.net/leixiaohua1020/article/details/17934487)的图，audio tag数据部分的第一个字节注明音频的编码类型、采样率、精度等信息。
![audio_data_tag](https://raw.githubusercontent.com/luotuo44/FlvExploere/master/images/audio_data_tag.jpg)

音频编码类型有下面几种：

|值|类型|
|:--:|:--:|
|0|Linear PCM, platform endian|
|1|ADPCM|
|2|MP3|
|3|Linear PCM, little endian|
|4|Nellymoser 16 kHz mono|
|5|Nellymoser 8 kHz mono|
|6|Nellymoser|
|7|G.711 A-law logarithmic PCM|
|8|G.711 mu-law logarithmic PCM|
|9|reserved|
|10|AAC|
|11|Speex|
|14|MP3 8 KHz|
|15|Device-specific sound|

音频采样率有下面几种：

|值|采样率|
|:--:|:--:|
|0|5.5 kHz|
|1|11 kHz|
|2|22 kHz|
|3|44 kHz|

精度只有2种，如下：

|值|精度|
|:--:|:--:|
|0|8位|
|1|16位|



