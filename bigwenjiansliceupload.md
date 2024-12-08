## 大文件分片上传

### 技术栈
-  react，spingboot，minio
### 实现思路
- 前端通过input标签选择文件，将文件分片，同时调用MD5算法计算文件哈希值，用作文件的唯一标识，将文件片循环上传至后端。
- 后端接收到分片文件，将文件分片上传至minio，用哈希值加文件名加分片序号命名，当最后一个分片接收成功，进行分片合并，从minio中拿到分片并使用文件流合并上传，成功上传后删除minio中多余的分片。
- 断点续传，快传，在接收的时候像minio中查找是否之前有传过，如果传过则skip，不执行后序操作，实现断点续传。
- 服务器启动时删除临时文件，同时设置定时间删除，访问minio中冗余分片堆积。
- 为了防止删除掉用户刚上传的片文件，每个片文件有一个过期时间，在clean的时候获取到当前时间，再减去片文件上次修改时间，若不大于24小时（或其他时间）则不执行删除操作。

### 前端代码
```tsx
import React, { useState } from "react";
import SparkMD5 from "spark-md5";

// 主组件
const FileUpload = () => {
  const [progress, setProgress] = useState(0); // 用于存储进度条的进度
  const [isUploading, setIsUploading] = useState(false); // 上传状态

  // 处理文件选择和分片上传
  async function handleFileChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    const result = createChunks(file, 1024*1024); // 将文件分片，并返回分片结果 1MB 104*1024 is 1MB
    console.log("Chunks:", result);
    const hs = await hash(result); // 计算文件哈希值，用作文件的唯一标识 

    console.log("File hash:", hs);

    let totalUploaded = 0; // 记录已上传的总字节数
    setIsUploading(true); // 开始上传

    for (let i = 0; i < result.length; i++) {
      const chunk = result[i];
      console.log("分片的大小：", chunk.size)
      const formData = new FormData();

      formData.append("file", chunk);
      formData.append("filename", file.name);
      formData.append("totalChunks", result.length.toString());
      formData.append("currentChunk", i.toString());
      formData.append("fileHash", hs as string); // 上传文件的哈希值（用于校验）

      // 上传分片并更新进度
      await uploadChunk(formData, (progress) => {
        totalUploaded += progress; // 增加当前上传的字节数
        const percentage = Math.min((totalUploaded / file.size) * 100, 100); // 计算上传百分比
        setProgress(percentage); // 更新进度条
      });
    }

    setIsUploading(false); // 上传完成
    // alert("File uploaded successfully!");
  }

  // 使用 XMLHttpRequest 上传文件分片，并实现进度条
  async function uploadChunk(formData: FormData, onProgress: (progress: number) => void) {
    return new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest();
      xhr.open("POST", "http://localhost:8080/upload", true);

      // 监听上传进度事件
      xhr.upload.onprogress = (e) => {
        if (e.lengthComputable) {
          const progress = e.loaded; // 已上传的字节数
          onProgress(progress); // 调用回调更新进度
        }
      };

      // 监听上传完成事件
      xhr.onload = () => {
        if (xhr.status === 200) {
          console.log("Chunk uploaded successfully");
          resolve(xhr.responseText);
        } else {
          console.log("Failed to upload chunk",xhr.statusText);
          reject(new Error("Failed to upload chunk"));
        }
      };

      xhr.onerror = (e) => {
        console.error("Error uploading chunk:", e);
        reject(new Error("Error uploading chunk"));
      };

      // 发送请求
      xhr.send(formData);
    });
  }

  // 创建文件分片
  function createChunks(file: any, chunkSize: any) {
    const result: Blob[] = [];
    for (let i = 0; i < file.size; i += chunkSize) {
      result.push(file.slice(i, i + chunkSize));
    }
    return result;
  }

  // 增量算法，计算文件哈希
  function hash(chunks: any) {
    return new Promise((resolve) => {
      function _read(i: number) {
        const spark = new SparkMD5();
        if (i >= chunks.length) {
          resolve(spark.end());
          return;
        }
        const blob = chunks[i];
        const reader = new FileReader();
        reader.onload = (e: any) => {
          const bytes = e.target.result;
          spark.append(bytes);
          _read(i + 1);
        };
        reader.readAsArrayBuffer(blob);
      }
      _read(0);
    });
  }

  return (
    <div style={styles.container}>
      <div style={styles.card}>
        <h1 style={styles.title}>File Upload</h1>
        <input
          type="file"
          onChange={handleFileChange}
          style={styles.fileInput}
        />
        {isUploading && <div style={styles.uploading}>Uploading...</div>}
        <div style={styles.progressWrapper}>
          <div style={styles.progressBarContainer}>
            <div
              style={{
                ...styles.progressBar,
                width: `${progress}%`,
              }}
            />
          </div>
          <p style={styles.progressText}>{Math.round(progress)}% Uploaded</p>
        </div>
      </div>
    </div>
  );
};

// 样式
const styles = {
  container: {
    display: "flex",
    justifyContent: "center",
    alignItems: "center",
    height: "100vh",
    backgroundColor: "#f5f5f5",
    fontFamily: "Arial, sans-serif",
  },
  card: {
    backgroundColor: "#fff",
    padding: "30px",
    borderRadius: "12px",
    boxShadow: "0 4px 8px rgba(0, 0, 0, 0.1)",
    width: "400px",
    textAlign: "center",
  },
  title: {
    fontSize: "24px",
    marginBottom: "20px",
    color: "#333",
  },
  fileInput: {
    padding: "10px",
    fontSize: "16px",
    borderRadius: "8px",
    border: "1px solid #ccc",
    marginBottom: "20px",
    width: "100%",
    cursor: "pointer",
    backgroundColor: "#f9f9f9",
  },
  uploading: {
    margin: "20px 0",
    fontSize: "16px",
    color: "#ff6600",
  },
  progressWrapper: {
    marginTop: "20px",
  },
  progressBarContainer: {
    backgroundColor: "#f3f3f3",
    borderRadius: "10px",
    height: "12px",
    overflow: "hidden",
  },
  progressBar: {
    height: "100%",
    backgroundColor: "#4caf50",
    borderRadius: "10px",
    transition: "width 0.2s ease-in-out",
  },
  progressText: {
    marginTop: "10px",
    fontSize: "14px",
    color: "#777",
  },
};

export default FileUpload;
```

### 后端代码
```java
package com.example.bookmanager.controller;

import io.minio.*;
import io.minio.errors.*;
import io.minio.messages.Item;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.time.ZonedDateTime;

@RestController
public class SliceUploadController {

    private String bucketName = "test-bucket";  // MinIO 桶的名称
    private String minioEndpoint = "http://localhost:9000";  // MinIO 地址
    private String minioAccessKey = "minioadmin";  // MinIO AccessKey
    private String minioSecretKey = "minioadmin";  // MinIO SecretKey

    // MinIO 客户端
    private MinioClient minioClient;

    public SliceUploadController() {
        try {
            minioClient = MinioClient.builder()
                    .endpoint(minioEndpoint)
                    .credentials(minioAccessKey, minioSecretKey)
                    .build();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @PostMapping("/upload")
    public String uploadChunk(@RequestParam("file") MultipartFile file,
                              @RequestParam("filename") String filename,
                              @RequestParam("totalChunks") int totalChunks,
                              @RequestParam("currentChunk") int currentChunk,
                              @RequestParam("fileHash") String fileHash) throws IOException {
        System.out.println("coming");

        // 判断 MinIO 中是否存在该桶
        try {
            boolean bucketExists = minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());
            if (!bucketExists) {
                // 如果桶不存在，则创建桶
                minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucketName).build());
            }
        } catch (MinioException e) {
            e.printStackTrace();
            return "Error while checking bucket existence.";
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }

        // 上传文件分片到 MinIO，使用 fileHash 作为一部分文件名，确保唯一性
        try {
            String objectName = fileHash + "-" + filename + "-chunk-" + currentChunk;  // 使用 fileHash 和 filename 来确保唯一性

            //实现断点续传,检索分片是否纯在
            boolean chunkExists = checkChunkExists(objectName);  // 检查分片是否已存在

            if (chunkExists) {
                return "Chunk already uploaded, skipping.";
            }


            minioClient.putObject(
                    PutObjectArgs.builder()
                            .bucket(bucketName)
                            .object(objectName)  // 存储对象名称（每个分片都有唯一名称）
                            .stream(file.getInputStream(), file.getSize(), -1)
                            .build()
            );

            // 检查是否上传了所有分片，合并分片
            if (currentChunk == totalChunks - 1) {
                // 合并分片到 MinIO
                mergeChunksAndUploadToMinIO(filename, totalChunks, fileHash);
            }

            // 手动删除临时文件
//            deleteTempFile(file);

            return "Chunk uploaded successfully";
        } catch (MinioException | IOException e) {
            e.printStackTrace();
            return "Error while uploading chunk.";
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
        return null;
    }

    // 检查分片是否已经上传
    private boolean checkChunkExists(String objectName) {
        try {
            StatObjectResponse stat = minioClient.statObject(
                    StatObjectArgs.builder()
                            .bucket(bucketName)
                            .object(objectName)
                            .build()
            );
            return stat != null;  // 如果返回结果不为 null，说明分片已上传
        } catch (MinioException | IOException e) {
            return false;  // 如果抛出异常，说明分片未上传
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
        return false;
    }


    // 删除 MinIO 上的文件
    @PostMapping("/remove")
    public String removeFileFromMinIO(@RequestParam("filename") String filename,  @RequestParam("fileHash") String fileHash) {
        String objectName = fileHash + "-" + filename;  // 使用 fileHash 和 filename 来确保唯一性
        try {
            minioClient.removeObject(
                    RemoveObjectArgs.builder()
                            .bucket(bucketName)
                            .object(objectName)
                            .build()
            );
            System.out.println("Successfully removed file from MinIO: " + objectName);
        } catch (MinioException | IOException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
        return null;
    }


    @Async
    public void mergeChunksAndUploadToMinIO(String filename, int totalChunks, String fileHash) {
        try {
            // 创建临时文件流，将所有分片合并到一个输出流
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

            // 合并每个分片
            for (int i = 0; i < totalChunks; i++) {
                String chunkObjectName = fileHash + "-" + filename + "-chunk-" + i;  // 使用 fileHash 作为分片标识
                // 从 MinIO 获取每个分片，并合并
                InputStream chunkStream = minioClient.getObject(
                        GetObjectArgs.builder()
                                .bucket(bucketName)
                                .object(chunkObjectName)
                                .build()
                );

                // 将当前分片写入到 byteArrayOutputStream
                byte[] buffer = new byte[1024];
                int bytesRead;
                while ((bytesRead = chunkStream.read(buffer)) != -1) {
                    byteArrayOutputStream.write(buffer, 0, bytesRead);
                }
                chunkStream.close();
            }

            // 将合并后的文件上传到 MinIO，使用 fileHash 来确保文件名唯一
            byte[] mergedData = byteArrayOutputStream.toByteArray();
            InputStream finalFileStream = new ByteArrayInputStream(mergedData);

            // 上传合并后的文件，文件名包含 fileHash 和原始文件名
            minioClient.putObject(
                    PutObjectArgs.builder()
                            .bucket(bucketName)
                            .object(fileHash + "-" + filename)  // 目标文件名
                            .stream(finalFileStream, mergedData.length, -1)
                            .build()
            );

//             合并完成后，可以删除分片
            for (int i = 0; i < totalChunks; i++) {
                String chunkObjectName = fileHash + "-" + filename + "-chunk-" + i;  // 使用 fileHash 来删除分片
                minioClient.removeObject(
                        RemoveObjectArgs.builder()
                                .bucket(bucketName)
                                .object(chunkObjectName)
                                .build()
                );
            }

            System.out.println("File successfully merged and uploaded to MinIO.");

        } catch (IOException | MinioException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
    }


    @PostConstruct
    public void init() {
        cleanUpChunks();  // 在应用启动时立即执行一次清理
    }

    // 每天凌晨 2 点执行清理任务
    @Scheduled(cron = "0 0 2 * * ?")
    @Async
    public void cleanUpChunks() {
        // 获取桶中的所有对象并删除符合条件的分片文件
        System.out.println("Cleaning up chunk files...");

        // 获取当前时间
        long currentTime = System.currentTimeMillis();

        // 使用 ListObjectsArgs 列出桶中的对象
        minioClient.listObjects(ListObjectsArgs.builder().bucket(bucketName).build()).forEach(result-> {
            Item item= null;
            try {
                item = result.get();
            } catch (ErrorResponseException | InsufficientDataException | InternalException | InvalidKeyException | InvalidResponseException | IOException | NoSuchAlgorithmException | ServerException | XmlParserException e) {
                e.printStackTrace();
            }
            String objectName = item.objectName();  // 获取对象名
            if (objectName.matches(".*-chunk-\\d+$")) { //使用正则表达式，删除以 -chunk- 后跟数字结尾的文件
                // 获取文件上传时间，假设文件的元数据包含上传时间
                // 获取文件的上传时间，使用 ZonedDateTime
                ZonedDateTime lastModified = item.lastModified();

                // 获取文件上传时间的毫秒时间戳
                long lastModifiedMillis = lastModified.toInstant().toEpochMilli();
                if (currentTime - lastModifiedMillis > 24 * 60 * 60 * 1000L) {
                    try {
                        // 删除符合条件的分片文件
                        minioClient.removeObject(RemoveObjectArgs.builder()
                                .bucket(bucketName)
                                .object(objectName)
                                .build());
                        System.out.println("Deleted chunk file: " + objectName);
                    } catch (MinioException | IOException e) {
                        e.printStackTrace();
                    } catch (NoSuchAlgorithmException e) {
                        e.printStackTrace();
                    } catch (InvalidKeyException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
    }

}


```