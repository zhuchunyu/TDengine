---
title: Data Compression
slug: /inside-tdengine/data-compression
---

import Image from '@theme/IdealImage';
import imgCompress from '../assets/data-compression-01.png';

Data compression is a technology that reorganizes and processes data using specific algorithms without losing effective information, aiming to reduce the storage space occupied by data and improve data transmission efficiency. TDengine employs this technology during the storage and transmission of data to optimize the use of storage resources and accelerate data exchange speed.

## Storage Compression

TDengine utilizes a columnar storage technology in its storage architecture, meaning that data is continuously stored by columns in the storage medium. This contrasts with traditional row-based storage, which stores data continuously by rows. Columnar storage, combined with the characteristics of time-series data, is particularly well-suited for handling stable time-series data.

To further enhance storage efficiency, TDengine employs differential encoding technology. This technique stores data by calculating the differences between adjacent data points rather than directly storing the raw values, significantly reducing the amount of information needed for storage. After differential encoding, TDengine also applies general compression techniques for secondary compression to achieve a higher compression ratio.

For stable time-series data collected from devices, the compression effect of TDengine is particularly notable, with compression rates typically reaching below 10%, and even higher in certain cases. This efficient compression technology saves users a substantial amount of storage costs while improving data storage and access efficiency.

### Primary Compression

Once time-series data is collected from devices, each collection device is modeled as a subtable according to TDengine's data modeling rules. Thus, all time-series data generated by a device is recorded in the same subtable. During the storage process, data is chunked for block storage, with each data block containing data from only one subtable. The compression operation is also performed on a block basis, compressing each column of data in the subtable, with the compressed data still stored in blocks on the hard drive.

The stability of time-series data is one of its primary characteristics; for example, atmospheric temperature and water temperature collected typically fluctuate within a certain range. This characteristic allows for re-encoding of the data, applying appropriate encoding techniques based on different data types to achieve maximum compression efficiency. The following describes various compression methods for different data types.

- Timestamp Type: Since the timestamp column usually records the moments when devices continuously collect data and the collection frequency is fixed, only the differences between adjacent time points need to be recorded. Since the differences are usually small, this method saves more storage space compared to directly storing the original timestamps.
- Boolean Type: Boolean values are represented by a single bit, allowing one byte to store 8 boolean values. Compact encoding significantly reduces storage space.
- Numeric Type: Numeric data generated by IoT devices, such as temperature, humidity, atmospheric pressure, vehicle speed, and fuel consumption, typically exhibit small values that fluctuate within a certain range. For this type of data, zigzag encoding is uniformly used. This technique maps signed integers to unsigned integers, shifting the most significant bit of the integer's complement to a lower position, while inverting all bits except the sign bit for negative numbers and keeping positive numbers unchanged. This method focuses valid data bits together while increasing the number of leading zeros, resulting in better compression effects during subsequent compression steps.
- Floating Point Type: For float and double types of floating point numbers, the delta-delta encoding method is used.
- String Type: String data utilizes dictionary compression algorithms to replace frequently occurring long strings in the original data with shorter identifiers, thus reducing the length of stored information.

### Secondary Compression

After performing dedicated compression for specific data types, TDengine further applies general compression techniques, treating data as indistinguishable binary data for secondary compression. Compared to primary compression, secondary compression focuses on eliminating information redundancy between data blocks. This dual compression technology emphasizes both local data streamlining and overall data overlap elimination, complementing each other to achieve ultra-high compression ratios in TDengine.

TDengine supports various compression algorithms, including LZ4, ZLIB, ZSTD, and XZ, allowing users to flexibly balance between compression ratio and write speed according to specific application scenarios and requirements, choosing the most suitable compression scheme.

### Lossy Compression

TDengine provides both lossless and lossy compression modes for floating point type data. The precision of floating point numbers is usually determined by the number of digits after the decimal point. In some cases, the precision of floating point numbers collected by devices is high, but the actual application may focus on lower precision; in such cases, lossy compression can effectively save storage space. The lossy compression algorithm in TDengine is based on predictive models, where the core idea is to use the trends of previous data points to predict the trends of subsequent data points. This algorithm can significantly improve compression ratios, outperforming lossless compression. The name of the lossy compression algorithm is TSZ.

## Transmission Compression

TDengine offers compression functionality during data transmission to reduce network bandwidth consumption. When transmitting data from clients (such as taosc) to the server using native connections, configuring for compressed transmission can save bandwidth. In the configuration file taos.cfg, the compressMsgSize option can be set to achieve this goal. The configurable values are as follows:

- 0: Indicates disabling compression transmission.
- 1: Indicates enabling compression transmission, but only for packets larger than 1KB.
- 2: Indicates enabling compression transmission for all packets.

When using RESTful and WebSocket connections to communicate with taosAdapter, taosAdapter supports industry-standard compression protocols, allowing the connection side to enable or disable compression during transmission based on industry-standard protocol. The specific implementation methods are as follows:

- RESTful Interface Using Compression: The client specifies the Accept-Encoding in the HTTP request header to inform the server of the acceptable compression types, such as gzip, deflate, etc. The server will specify the compression algorithm used in the Content-Encoding header when returning results and return compressed data.
- WebSocket Interface Using Compression: The implementation of compression in WebSocket connections can refer to the WebSocket protocol standard document RFC7692 for details.
- The data backup migration tool taosX can also enable compressed transmission between taosX and taosX Agent. In the agent.toml configuration file, set the compression switch option to compression=true to enable the compression feature.

## Compression Process

The figure below illustrates the compression and decompression processes of TDengine during the entire transmission and storage process of time-series data, providing a better understanding of the entire handling process.

<figure>
<Image img={imgCompress} alt="Compression and decompression process"/>
<figcaption>Figure 1. Compression and decompression process</figcaption>
</figure>
