---
layout: post
title:  "Chrome Headless build"
date:   2016-08-23 20:40:56 -0500
categories: chrome
---
## Intro
Back in June Sam Saccone had a post about Chrome Headless that seemingly was loved by many with a plethora of likes and re-tweets.
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Headless Chrome is coming so soon 🎪!!! Say goodbye to the myriad of hacks to run Chrome and phantom on CI. 🌊<a href="https://t.co/ucDBbgpBO0">https://t.co/ucDBbgpBO0</a></p>&mdash; Sam Saccone (@samccone) <a href="https://twitter.com/samccone/status/739166801427210240">June 4, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

With Chrome Headless there are a multitude of opportunities for testing but my goals were a little different.  I wanted to create a way to capture screenshots and generate PDF's.

## Building Chrome headless
Building Chrome is not short process. Make sure that you read all the steps before you begin so you don't have to start over.

1. Create an Ubuntu EC2 on AWS
There are are a few changes we need to make to the standard Ubuntu image before it is usable for building Chrome.

  1. Select the standard Ubuntu image.
![EC2]({{ site.url }}/assets/headless-assets/EC2.png)

  2. Select a large instance type. I usually go for `c3.4xlarge`. You need a machine with at least 16gb of memory.
![EC2 Type]({{ site.url }}/assets/headless-assets/EC2Type.png)

  3. The standard storage configuration is not acceptable and needs to be adjusted.
![Storage]({{ site.url }}/assets/headless-assets/Storage.png)

  4. I adjust the main storage to have 250gb. This is to make sure that there is enough space for any dependencies.
![EditedStorage]({{ site.url }}/assets/headless-assets/EditedStorage.png)

2. SSH to the instance `ssh -i ~/.ssh/{PemKey}.pem ubuntu@{MachineDNS}`

3. Switch user `sudo su`

4. Install git `apt-get install -y git`

5. Clone the [Google Depot][depot] tools `git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git`

6. Add Depot Tools to your bash profile `echo "export PATH=$PATH:/home/ubuntu/depot_tools" >> ~/.bash_profile`

7. Source the bash profile `source ~/.bash_profile`

8. Make a Chromium directory and enter it. `mkdir Chromium && cd Chromium`

9. Fetch Chromium source with no history. This will keep the download size down. `fetch --no-history chromium`

10. Enter the src directory `cd src`

11. Build the Chromium dependencies `./build/install-build-deps.sh --no-prompt`.

12. You will be prompted to accept a EULA for fonts. Accept the agreement.

13. Create an output directory `mkdir -p out/Headless`

14. Create a build arguments file

  1. `echo 'import("//build/args/headless.gn")' > out/Headless/args.gn`

  2. `echo 'is_debug = false' >> out/Headless/args.gn`

15. Generate Ninja Build files `gn gen out/Headless`

16. Kick off the build `ninja -C out/Headless headless_shell`

17. Verify the build was successful. `out/Headless/headless_shell http://goat.com`

18. Tar the build `tar -zcvf ChromeHeadless.tar.gz out/Headless`

## Copy the tarball to your local machine.
1. Use SCP to copy the tarball `scp -i ~/.ssh/{PemKey}.pem ubuntu@{MachineDNS}:/home/ubuntu/src/ChromiumHeadless.tar.gz ~/Desktop/ChromiumHeadless.tar.gz`

## Node
I am a huge fan of Typescript as such my Node server is written in Typescript.

{% highlight javascript %}
let express = require('express');
let fs = require('fs');
let Chrome = require('chrome-remote-interface');
let spawn = require('child_process').spawn;
let PDFDocument = require('pdfkit');
// The second argument when starting node is the location of the headless_shell binary.
if (process.argv[ 2 ] === undefined) {
  throw Error('No headless binary path provided.');
}

// We need to spawn Chrome headless with some parameters, one of which is the debug port.
// I am hard coding window size because I want the screenshot to be a specific size.
// Note: 1280x1696 is the default pixel resolution for a Letter sheet of paper.
spawn(process.argv[ 2 ], ['--no-sandbox', '--remote-debugging-port=9222','--window-size=1280x1696' ]);

let app = express();

// Just to demonstrate the app working fetch on root of the app causes the PDF to be generated.
app.get('/', (req, res)=> {
  Chrome.New(function () {
    Chrome(chromeInstance => {
      chromeInstance.Page.loadEventFired(takeScreenshot.bind(null, chromeInstance, res, Date.now()));
      chromeInstance.Page.enable();
      chromeInstance.once('ready', ()=> {
        chromeInstance.Page.navigate({url: 'https://addyosmani.com/'});
      })
    });
  });
});


function takeScreenshot(instance, response, startTime) {
  let screenshot = function (v) {
    let filename = `screenshot-${Date.now()}`;
    fs.writeFileSync(filename + '.png', v.data, 'base64');

    console.log(`Image saved as ${filename}.png`);

    let imageEnd = Date.now();

    console.log('image success in: ' + (+imageEnd - +startTime) + "ms");
    // Using PDFKit to create the actual PDF document
    let doc = new PDFDocument({
      margin:0,
      size:'letter'
    });
    // This width is in points not pixels.
    doc.image(filename + '.png', 0,0, {width:612});
    // I wanted the file response to trigger a download rather than loading in the page.
    response.setHeader('Content-Type', 'application/pdf');
    response.setHeader('Content-Disposition', 'attachment; filename=' + filename + '.pdf');
    // Pipe the document into the response and close the stream when it's completed.
    doc.pipe(response);

    let docEnd = Date.now();
    doc.end();

    let endTime = Date.now();
    console.log('pdf success in: ' + (+docEnd - +startTime) + "ms");
    console.log('success in: ' + (+endTime - +startTime) + "ms");
    // Kill the tab we are using to save on memory.
    instance.close();
  };
  // This is the magic here.
  instance.Page.captureScreenshot().then(screenshot);
}

app.listen(3000, function () {
  Chrome.Version().then(version => {
    console.log(version)
  });
  console.log('Export app running on 3000!');
});
{% endhighlight %}

## Docker
Part of being able to run our export service headless is actually deploying it headlessly.  To do this we are going to use Docker.
Below is the Dockerfile used to build a docker machine with all the dependencies necessary to run Chromium and Node.

{% highlight docker %}
FROM ubuntu:16.04

# Install.
RUN \
    apt-get update && \
    apt-get install sudo && \
    sed -i 's/# \(.*multiverse$\)/\1/g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get -y upgrade && \
    apt-get install -y build-essential && \
    apt-get install -y software-properties-common && \
    apt-get install -y byobu curl git htop man unzip vim wget && \
    rm -rf /var/lib/apt/lists/* && \
    curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash - && \
    sudo apt-get install -y nodejs && \
    sudo apt-get install -y libnss3 && \
    sudo apt-get install -y libgtk2.0-0 libgdk-pixbuf2.0-0 libfontconfig1 libxrender1 libx11-6 libglib2.0-0 libxft2 libfreetype6 libc6 zlib1g libpng12-0 libstdc++6-4.8-dbg-arm64-cross libgcc1

COPY ./node_modules /root/export-app/node_modules
ADD ./app.js /root/export-app/app.js
ADD ./package.json /root/export-app/package.json
ADD ./ChromiumHeadless.tar.gz /root/export-app

WORKDIR /root/export-app

EXPOSE 8080 3000

CMD ["node", "app.js", "/root/export-app/chromium/src/out/Debug/headless_shell"]
{% endhighlight %}


## Thanks
None of this could have been done without amazing help from a large group of people:
[Sam Saccone][sams], [Andrea Cardaci][andrea], [Sami Kyöstilä][sami], [Paul Lewis][paull], [Amin Moshgabadi][amin], [Eric Seckler][eric]

[eric]: https://www.linkedin.com/in/ericseckler
[amin]: https://www.linkedin.com/in/amoshg
[paull]: https://aerotwist.com/
[sami]: http://www.unrealvoodoo.org/
[andrea]: https://cyrus-and.github.io/
[sams]: http://github.com/sami
[depot]: https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up