<template>
  <div id="app">
    <img :src="url" height="200"/>
    <button @click="createFaceCollection">Create face collection</button>
    <button @click="dropFaceCollection">Drop face collection</button>

    <br>
    UploadFace
    <input type="file" @change="uploadFace($event)"/>

    FindFace
    <input type="file" @change="findFace($event)"/>
  </div>
</template>

<script>
  import AWS from 'aws-sdk';
  AWS.config.update({
    region: "us-east-1",
    accessKeyId: "...",
    secretAccessKey: "...",
  });

  export default {
    data() {
      const bucketname = 'rekognition-face-verification';
      return {
        url: '',
        CollectionId: 'MyCollection',
        bucketname,
        rekognition: new AWS.Rekognition(),
        s3: new AWS.S3({
          apiVersion: '2006-03-01',
          params: { Bucket: bucketname }
        }),
      }
    },
    name: 'HelloWorld',
    props: {
      msg: String
    },
    methods: {
      createFaceCollection() {
        const { CollectionId } = this;
        this.rekognition.createCollection({ CollectionId }, function(err, data) {
          if (err) console.log(err, err.stack); // an error occurred
          console.log(data);           // successful response
        });
      },
      dropFaceCollection() {
        const { CollectionId } = this;
        this.rekognition.deleteCollection({ CollectionId }, function(err, data) {
          if (err) console.log(err, err.stack); // an error occurred
          console.log(data);           // successful response
        });
      },
      uploadFace(e) {
        const [ file ] = e.target.files;
        const { CollectionId } = this;

        let fileBuffer;

        const reader = new FileReader();
        reader.onload = (e) => {
          fileBuffer = reader.result;

          this.rekognition.searchFacesByImage({
            CollectionId,
            Image: {
              Bytes: fileBuffer
            },
            MaxFaces: 1,
          }, (err, data) => {
            if (err) console.log(err);
            console.log(data)
          });

          this.s3.upload({
            Body: file,
            Key: 'TEST_PHOTO',
          }, (err, data) => {
            if (err) {
              return console.log(err)
            }
            console.log(data);
            const { Bucket, Key } = data;

            this.s3.getObject({ Bucket, Key },
              (error, data) => {
                if (error) {
                  return console.log(error);
                }
                console.log(data);
//              this.DetectFaces(data.Body);

                const blob = new Blob([data.Body], {'type': 'image/png'});
                this.url = URL.createObjectURL(blob);
//___

                this.rekognition.indexFaces({
                  CollectionId,
                  Image: {
//                    S3Object: {
//                      Bucket,
//                      Name: Key,
//                    }
                    Bytes: fileBuffer
                  },
                  MaxFaces: 1,
                  QualityFilter: 'AUTO',
                  DetectionAttributes: ['ALL']
                }, (err, data) => {
                  if (err) console.log(err, err.stack); // an error occurred
                  console.log(data);
                });
              });
          })
        };
        reader.readAsArrayBuffer(file);
      },
      findFace(e) {
        const [ file ] = e.target.files;
        const { CollectionId } = this;

        let fileBuffer;

        const reader = new FileReader();
        reader.onload = (e) => {
          fileBuffer = reader.result;

          this.rekognition.searchFacesByImage({
            CollectionId,
            Image: {
              Bytes: fileBuffer
            },
            MaxFaces: 1,
          }, (err, data) => {
            if (err) console.log(err);
            console.log(data)
          });
        };
        reader.readAsArrayBuffer(file);
      },
    }
  }
</script>

