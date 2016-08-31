# rgw s3 libs3\(C\/C++\)SDK使用手册

## 概要：

rgw兼容S3接口，libs3支持用来支持C\/C++访问S3的SDK，这里主要介绍该库提供的基本方法及使用实例，一下方法都声明于libs3.h文件中。

1. libs3数据结构

  1.  S3ListServiceHandler

    ```
    typedef struct S3ListServiceHandler
    {
        /**
         * responseHandler provides the properties and complete callback
         **/
        S3ResponseHandler responseHandler;
        /**
         * The listServiceCallback is called as items are reported back from S3 as
         * responses to the request
         **/
        S3ListServiceCallback *listServiceCallback;
    } S3ListServiceHandler;
    ```

  2.  S3ResponseHandler
    ```
    /**
     * An S3ResponseHandler defines the callbacks which are made for any
     * request.
     **/
    typedef struct S3ResponseHandler
    {
        /**
         * The propertiesCallback is made when the response properties have
         * successfully been returned from S3.  This function may not be called
         * if the response properties were not successfully returned from S3.
         **/
        S3ResponsePropertiesCallback *propertiesCallback;
        
        /**
         * The completeCallback is always called for every request made to S3,
         * regardless of the outcome of the request.  It provides the status of
         * the request upon its completion, as well as extra error details in the
         * event of an S3 error.
         **/
        S3ResponseCompleteCallback *completeCallback;
    } S3ResponseHandler;
    ```

  3.  S3Protocol
    ```
    typedef enum
    {
        S3ProtocolHTTPS                     = 0,
        S3ProtocolHTTP                      = 1
    } S3Protocol;
    ```


2.  访问方式
  1. Encryption & Unencrypted
    通过指定S3Potocol来决定是否对请求加密（http\/https） 
  2. Path URL & Virtual Host

3. libs3函数接口使用
  1.  Initialize
    ```
    S3Status S3_initialize(const char *userAgentInfo, int flags,
                           const char *defaultS3HostName)
    ```

  2.  Service Ops
    ```
    void S3_list_service(S3Protocol protocol, const char *accessKeyId, const char *secretAccessKey, 
                         const char *securityToken, const char *hostName, S3RequestContext *requestContext,
                          const S3ListServiceHandler *handler, void *callbackData)
    ```

  3.  Bucket Ops
    * Create Bucket
      ```
       void S3_create_bucket(S3Protocol protocol, const char *accessKeyId,
                            const char *secretAccessKey, const char *securityToken,
                            const char *hostName, const char *bucketName,
                            S3CannedAcl cannedAcl, const char *locationConstraint,
                            S3RequestContext *requestContext,
                            const S3ResponseHandler *handler, void *callbackData)
      ```

    *  Delete Bucket
      ```
      void S3_delete_bucket(S3Protocol protocol, S3UriStyle uriStyle,
                            const char *accessKeyId, const char *secretAccessKey,
                            const char *securityToken, const char *hostName, 
                            const char *bucketName, S3RequestContext *requestContext,
                            const S3ResponseHandler *handler, void *callbackData)
      ```

    *  Get Bucket
      ```
      void S3_list_bucket(const S3BucketContext *bucketContext, const char *prefix,
                          const char *marker, const char *delimiter, int maxkeys,
                          S3RequestContext *requestContext,
                          const S3ListBucketHandler *handler, void *callbackData)
      ```

    *  Test Bucket
      ```
      void S3_test_bucket(S3Protocol protocol, S3UriStyle uriStyle,
                          const char *accessKeyId, const char *secretAccessKey,
                          const char *securityToken, const char *hostName,
                          const char *bucketName, int locationConstraintReturnSize,
                          char *locationConstraintReturn,
                          S3RequestContext *requestContext,
                          const S3ResponseHandler *handler, void *callbackData)
      ```


  4. Object Ops

  5.  Deinitialize
    ```
    S3_deinitialize()
    ```


4.  libs3使用实例
  ```
  #include <stdlib.h>
  #include <stdio.h>
  #include <string.h>
  #include <strings.h>
  #include <sys/stat.h>
  #include <sys/types.h>
  #include <time.h>
  #include <getopt.h>
  #include <ctype.h>
  #include "libs3.h"

  static S3Status responsePropertiesCallback
      (const S3ResponseProperties *properties, void *callbackData)
  {
      return S3StatusOK;
  }

  static void responseCompleteCallback(S3Status status,
                                       const S3ErrorDetails *error,
                                       void *callbackData)
  {
  }

  static S3Status listServiceCallback(const char *ownerId,
                                      const char *ownerDisplayName,
                                      const char *bucketName,
                                      int64_t creationDate, void *callbackData)
  {
      bool *header_printed = (bool*) callbackData;
      if (!*header_printed) {
          *header_printed = true;
          printf("%-22s", "       Bucket");
          printf("  %-20s  %-12s", "     Owner ID", "Display Name");
          printf("\n");
          printf("----------------------");
          printf("  --------------------" "  ------------");
          printf("\n");
      }
      printf("%-22s", bucketName);
      printf("  %-20s  %-12s", ownerId ? ownerId : "", ownerDisplayName ? ownerDisplayName : "");
      printf("\n");
      return S3StatusOK;
  }


  int main(int argc, char** args)
  {
      S3Protocol protocolG = S3ProtocolHTTP;
      const char* accessKeyIdG = getenv("S3_ACCESS_KEY_ID");
      const char* secretAccessKeyG = getenv("S3_SECRET_ACCESS_KEY");
      S3Status status;
      const char* hostname = getenv("S3_HOSTNAME");
      if ((status = S3_initialize("s3", S3_INIT_ALL, hostname)) != S3StatusOK)
      {
          fprintf(stdout, "Failed to initialize libs3: %s\n", S3_get_status_name(status));
          exit(-1);
      }
      S3ResponseHandler responseHandler = { &responsePropertiesCallback,
                                            &responseCompleteCallback
                                          };
      S3ListServiceHandler listServiceHandler = { responseHandler,
                                                  &listServiceCallback
                                                 };
      bool data = false;
      //---------------------------list all owned buckets-----------------//
      S3_list_service(protocolG, accessKeyIdG, secretAccessKeyG, 0, 0, 0,
                          &listServiceHandler, &data);

      //-----------------------------create bucket------------------------//
      const char* bucketName = "unitedstack-bucket";
      S3_create_bucket(protocolG, accessKeyIdG, secretAccessKeyG, 0,
                           0, bucketName, S3CannedAclPrivate, 0, 0,
                           &responseHandler, 0);
      S3_deinitialize();
      return 0;
  }
  ```


