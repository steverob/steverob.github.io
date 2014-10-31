---
layout: post
title: "Outsourcing image processing"
date: 2014-10-31 01:06:26 +0530
comments: true
categories: Rails, Scaling
---

One of our clients at Spritle is a startup that sells photos taken at various events. The Rails app we have built and maintaining for them involves photographers uploading large numbers of photos (each photographer uploads around 500-1000 images per album and we have lots of photographers, albums & events!) at a time and processing these images into three versions of different dimensions. One of these versions also needs a watermark to be placed on it. We had a very basic setup to do all this work but it was not good enough and subsequently we offloaded the heavyligting to a cloud service. But before that let me explain how we were doing things before offloading the processing.
<!--more-->
So when photographers upload photos from the application, the photos are not sent to our server. Instead they are uploaded directly to our S3 bucket and the URL of these photos are continuously posted to our application as each photo completes uploading. Our Rails app upon receiving this URL saves it in the database and queues a job for our [Sidekiq](https://github.com/mperham/sidekiq) workers to perform. The Sidekiq job simply passes the S3 URL of our photo to Photo model's `remote_resource_url=` method which is provided by [Carrierwave](https://github.com/carrierwaveuploader/carrierwave). This method downloads the image from the remote location - S3 in our case, processes it, generating the versions we need, uploads these three versions to S3 again and updates the `resource` attribute of the photo with the final location of the image on S3. Carrierwave is clever in that it stores only the S3 key of the original version (unprocessed) in the Photo model. Since it names the other versions as `<version_name>_image_name` it can easily provide us with URLs for other versions if needed. 

This setup worked perfectly for us. But the problem was the throughput. Each image took nearly 4 minutes to process which was unacceptable given that we would have photographers uploading thousands of images within couple of hours on some days and then would have to wait for several hours for all their images to be processed. This had a bad impact on sales as our aim is to get the event photos available to the buyers as soon as possible once the events are over. We tried increasing the number of sidekiq workers and that improved the throughput a bit but we could never sustain doing something like that. Sure we could add couple more EC2 instances into the mix and speed things up but economically it was not a desireable path to take for the startup. This is something we are planning to do in the near future and it involves using Elixir on the awesome Erlang VM. But for now we needed a more cost effective and headache free solution.

We took the decision to go with an external service to take care of the image processing work. Within few minutes of research I found two promising services. They were [Cloudinary](http://cloudinary.com/) and [Blitline](http://www.blitline.com/). Cloudinary is incredibly sophisticated with lots of features including tons of image manipulation options, enhancements, etc and also provides a CDN for our images. They even provide URL-based manipulations where we can encode the operations to be performed on an image as part of the URL itself and the image is served after the operations are applied. But there is a catch with Cloudinary. Cloudinary provides its own storage and there is no way to tell Cloudinary to store the processed images in our own S3 bucket unless you go with their Advanced plan. There are options to backup images to our S3 and also we could use their API to move images to our bucket. But then this aint a straight forward solution. 

Another problem for us was the storage limits Cloudinary had put in place. Even if you take their Advanced plan which costs $200/m you get only 100 GB of storage. For me Blitline sounded much more interesting.

With Blitline all you need to do is make an API call to it telling where to get the original image (available in our S3 bucket), what transformations need to be performed and where the results need to be stored. Blitline will respond after completing these operations with the URLs of the processed images. You can do multiple transformations of the original image and Blitline will respond with URLs for all those versions. For us Blitline's flexible API and the pricing model which is based on the hours of processing used were a big win. 

Let's look at a basic example of a job submitted to Blitline. This one is from their API docs:

{% codeblock basic_job.ruby %}
{
  "application_id" => "YOUR_APP_ID",
  "src" => "http://www.google.com/logos/2011/yokoyama11-hp.jpg",
  "functions" => [{
    "name" => "blur",
    "save" => { "image_identifier" => "MY_CLIENT_ID" }
  }]
}
{% endcodeblock %}

The response to this simple job is this:

{% codeblock response.json %}
{
  "results":
  {
    "images":[{
                 "image_identifier": "MY_CLIENT_ID",
                 "s3_url": "https://dev.blitline.s3.amazonaws.com/2011111513/1/fDIFJQVNlO6IeDZwXlruYg.jpg"
             }],
    "job_id": "4ec2e057c29aba53a5000001"
  }
}
{% endcodeblock %}

Here is a slightly more complex job where we tell Blitline where to find our original image and after resizing, where to store it:

{% codeblock s3_example.ruby %}
{
  "application_id"=> ENV['BLITLINE_APP_ID'],
  "src" => "original_photo_url",
  "functions" => [{
    "name"=> "resize_to_fit",
    "params" => {
      "height"=>400
    },
    "save" => {
      "image_identifier" => "large",
      "s3_destination" => {
        "bucket" => ENV['S3_BUCKET'],
        "key" => "key_for_object",
        "headers"=> {
          "x-amz-grant-read" =>"",
          "x-amz-meta-foo" => "authenticated-read"
        }
      }
    }
  }]
}
{% endcodeblock %}

The response to this is also similar to the previous one but it differs in the fact that now the resultant image will be in our S3 bucket. 

One of the nicer features of Blitline is how it allows us to nest functions so that the resultant image of the parent is used for processing the nested functions. Here is a job demonstrating just that. Infact this is the job we are performing for our service.

{% codeblock nested_functions.ruby %}
{
  "application_id"=> ENV['BLITLINE_APP_ID'],
  "src" => @photo.original_resource_url,
  "pre_process" => {
    "move_original" => {
      "s3_destination" => {
        "bucket" => ENV['S3_BUCKET'],
        "key" => "key_for_object",
        "headers"=> {
          "x-amz-grant-read" =>"",
          "x-amz-meta-foo" => "authenticated-read"
        }
      }
    }
  },
  "functions" => [{
    "name"=> "resize_to_fit",
    "params" => {
      "height"=>400
    },
    "save" => {
      "image_identifier" => "large",
      "s3_destination" => {
        "bucket" => ENV['S3_BUCKET'],
        "key" => "key_for_object",
        "headers"=> {
          "x-amz-grant-read" =>"",
          "x-amz-meta-foo" => "authenticated-read"
        }
      }
    },
    "functions" =>[
      {
        "name"=> "resize_to_fit",
        "params" => {
          "height"=>120
        },
        "save" => {
          "image_identifier" => "small",
          "s3_destination" => {
            "bucket" => ENV['S3_BUCKET'],
            "key" => "key_for_object",
            "headers"=> {
              "x-amz-grant-read" =>"",
              "x-amz-meta-foo" => "authenticated-read"
            }
          }
        }
      },
      {
        "name"=> "resize_to_fit",
        "params" => {
          "height"=>300
        },
        "functions" => [{
          "name" => "composite",
          "params" => {
            "src"=> "url_of_watermark_image",
            "gravity"=> "CenterGravity"
          },
          "save" => {
            "image_identifier" => "social",
            "s3_destination" => {
              "bucket" => ENV['S3_BUCKET'],
              "key" => "key_for_object"
            }
          }
        }]
      }
    ]
  }]
}

{% endcodeblock %}

In the above job we have this key called _pre_process_. This allows us to specify the transformations that need to be performed on the original source image. Here we just instruct it to move it to our S3 bucket. Followed by that we have a function that generates a small version of the original (400px high) and stores it in our bucket. Within this function we nest two more functions. One of them generates a much smaller version and the other generates an image with height of 300px and a watermark on it. These are stored in our S3 bucket as well.

Super easy and flexible right? :) Blitline provides tons more of manipulation options and other features. Also they provide post-back feature whereby we need not keep polling them to see if the job is complete. Instead they will call us at the specified end point once processing is done! Just have a look at their website to find out more. Although their documentation is not as good as Cloudinary's, I thought it was quite adequate. Their support was also super responsive when I contacted them with issues. They also claim to be cash-positive since 2011 in their website which is also a very good and important thing.

Anyway, coming back to our project, after integrating Blitline with our Rails application the throughput which was at around 12.5 images / min rose to over 100 images / min. :) This inturn meant that the images take in events were made available for purchasing with a very short while and which in turn means our client is happy and making profit! :)