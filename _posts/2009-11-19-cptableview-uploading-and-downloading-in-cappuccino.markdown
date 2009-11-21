---
layout: post
title: "CPTableView, Uploading and Downloading in Cappuccino"
---

*Note: This post might currently be factually incorrect and contain faulty code. Hopefully this will be reviewed by someone more competent than me shortly and any errors will be corrected. As it is, this code does not work in Firefox and IE is untested.*

As a project to learn how to use Cappuccino and specifically learn a bit more about CPTableView as well as how to send and receive files from the user I created this small uploading application.

![CPUploadDemo](/content/cpuploaddemo.png "The result of this tutorial")

To spread the joy of Cappuccino I want to make this into a tutorial. To kick it off, we need some sort of backend for the file handling, since I'm comfortable with Ruby on Rails thats what I chose. The backend is a really simple application using the paperclip plugin.

For my UploadsController.rb I have these three actions:
{% highlight ruby %}
class UploadsController < ApplicationController

  def index
    @uploads = Upload.all
    render :json => @uploads.to_json
  end

  def show
    upload = Upload.find(params[:id])
    send_file upload.attachment.path, :type => upload.attachment_content_type
  end

  def create
    @upload = Upload.new(params[:upload])
    if @upload.save
      render :text => @upload.to_json, :status => :created
    else
      render :text => @upload.errors.to_json, :status => :unprocessable_entity
    end
  end

end
{% endhighlight %}

And then in my Upload.rb model I use the paperclip plugin like this:
{% highlight ruby %}
class Upload < ActiveRecord::Base
  has_attached_file :attachment
end
{% endhighlight %}

And that's it! Backends are so simple with Cappuccino.

Now for the fun stuff. First of all, to upload a file we need some way to send the file to the server. To do this I have used [Ross Bouchers](http://github.com/boucher) original implementation of FileUpload.j found in [this gist](http://gist.github.com/11652 "FileUpload.j"), although there are several modifications of this so there might be better versions out there.

I'm not going to worry too much about the contents of this file but rather just care about how to use it. What we need to know is that when something happens with the button it sends some notifications, the notifications we care about are the following.

{% highlight objc %}
- (void)uploadButton: uploadButton didChangeSelection: selection
- (void)uploadButton: uploadButton didFinishUploadWithData: response
{% endhighlight %}

The first one is called when a file has been selected with the button and the second gets called when the upload is finished.

Let's look at setting up the application and interface now.

First of all, we import some necessary files and set up some instance variables so we can access them from anywhere within our AppController class.

{% highlight objc %}
import <Foundation/CPObject.j>
import "FileUpload.j"
import "FButtonView.j"

implementation AppController : CPObject
{
  CPTextField label;
  CPTableView tableView;
  CPArray files;
}
{% endhighlight %}
(For some reason my highlighting wont allow me to have @ in front of import and implementation but there should be.)

Now we set up the application interface and start sending off requests to load data.

{% highlight objc %}
- (void)applicationDidFinishLaunching:(CPNotification)aNotification
{
  // send off a request for the file-list
  request = [[CPURLRequest alloc] initWithURL:@"http://localhost:3000/uploads"]; ;
  [CPURLConnection connectionWithRequest:request delegate:self];
  
  // set up the window and create a variable to access the contentView a bit easier
  var theWindow = [[CPWindow alloc] initWithContentRect:CGRectMakeZero()  
                                    styleMask:CPBorderlessBridgeWindowMask],
  contentView = [theWindow contentView];

  // create the label and position it
  label = [[CPTextField alloc] initWithFrame:CGRectMakeZero()]
  [label setStringValue:@"Hello, upload something!"];
  [label setFont:[CPFont boldSystemFontOfSize:24.0]];
  [label sizeToFit];
  [label setCenter:CGPointMake(CGRectGetWidth([contentView frame])/2.0, 20)];
  [label setAutoresizingMask:CPViewMinXMargin | CPViewMaxXMargin];

  // add the label to the window
  [contentView addSubview:label];

  // create a CPScrollView that will contain the CPTableView
  var scrollView = [[CPScrollView alloc] initWithFrame:CGRectMake(
    CGRectGetWidth([contentView bounds])/2-150, 50.0, 300.0, 
    CGRectGetHeight([contentView bounds])-100)];
  [scrollView setAutohidesScrollers:YES];
  [scrollView setAutoresizingMask:CPViewMinXMargin | CPViewMaxXMargin | CPViewHeightSizable];

  // create the CPTableView
  tableView = [[CPTableView alloc] initWithFrame:[scrollView bounds]];
  [tableView setDataSource:self];
  [tableView setUsesAlternatingRowBackgroundColors:YES];
  
  // define the header color
  var headerColor = [CPColor colorWithPatternImage:[[CPImage alloc]
    initWithContentsOfFile:[[CPBundle mainBundle] pathForResource:@"button-bezel-center.png"]]];
  [[tableView cornerView] setBackgroundColor:headerColor];
  
  // add the filename column
  var column = [[CPTableColumn alloc] initWithIdentifier:@"Filename"];
  [[column headerView] setStringValue:"Filename"];
  [[column headerView] setBackgroundColor:headerColor];
  [column setWidth:280.0];
  [tableView addTableColumn:column];
  
  // add the downloadbutton column
  var downloadColumn = [[CPTableColumn alloc] initWithIdentifier:"Download"];
  [downloadColumn setWidth:20];
  [[downloadColumn headerView] setBackgroundColor:headerColor];
  [downloadColumn setDataView:[[FButtonView alloc] initWithFrame:CGRectMake( 0, 0, 20, 20 )]];
  [tableView addTableColumn:downloadColumn];
  
  // set the tableview as the documentview of scrollview
  [scrollView setDocumentView:tableView];
  
  // add the scrollView to the window
  [contentView addSubview:scrollView];
  
  // create and set up the upload button
  uploadButton = [[UploadButton alloc] initWithFrame: CGRectMakeZero()] ;
  [uploadButton setTitle:@"Select File"] ;
  [uploadButton setName:@"upload[attachment]"]
  [uploadButton setURL:@"http://localhost:3000/uploads/create"];
  [uploadButton setDelegate: self];
  [uploadButton setAutoresizingMask:CPViewMinXMargin | CPViewMaxXMargin | CPViewMinYMargin];
  [uploadButton setBordered:YES];
  [uploadButton sizeToFit];
  [uploadButton setCenter:CGPointMake(CGRectGetWidth([contentView bounds])/2.0,
    CGRectGetHeight([contentView bounds])-25)];
  
  // add the uploadbutton to the window
  [contentView addSubview:uploadButton];

  [theWindow orderFront:self];
}
{% endhighlight %}

Objective-J is quite verbose, so I wont spend much time explaining this as it seems pretty straightforward. The highlight here is the request sent off in the beginning and the creation of the UploadButton. The request grabs the file-list from the backend. For this we need a bit more logic and that is done in the connection notification.

When the request is completed, didReceiveData gets called and we create an object from the JSON returned by the backend. When this is done we need to reload the tableView so the new data is shown. A smart thing would be to implement the didFailWithError method as well, but I'll skip that right now.
{% highlight objc %}
- (void)connection:(CPURLConnection)aConnection didReceiveData:(CPString)data
{
  //get a javascript object from the json response
  files = CPJSObjectCreateWithJSON(data);
  [tableView reloadData];
}
{% endhighlight %}

Now to show this data in the table we need to implement the methods that get called on the datasource when the CPTableView loads the data.

{% highlight objc %}
// CPTableView datasource methods
- (int)numberOfRowsInTableView:(CPTableView)tableView
{
  return [files count];
}

- (id)tableView:(CPTableView)tableView objectValueForTableColumn:
    (CPTableColumn)tableColumn row:(int)row
{
  if([tableColumn identifier] == "Filename") {
    return [files objectAtIndex:row].upload.attachment_file_name;   
  } else {
    return [files objectAtIndex:row].upload.id;
  }
}
{% endhighlight %}

The first one just returns how many rows the tableview should contain and the second returns the data that the tablecolumn should contain. By default you can return a string to the tablecolumns dataview and it will write out that string. But if you looked carefully at the big chunk of code above you would have noticed that I set the dataview of the downloadColumn to an FButtonView, this is my own subclass of CPView that handles the displaying of a downloadbutton and the subsequent downloading.

Before we look at FButtonView we'll have a quick look at how we handle the upload events.
{% highlight objc %}
// UploadButton methods
- (void)uploadButton: uploadButton didChangeSelection: selection
{
  [uploadButton submit];
}
- (void)uploadButton: uploadButton didFinishUploadWithData: response
{
  [label setStringValue:@"File uploaded!"];
  [label sizeToFit];
  files = [files arrayByAddingObject:CPJSObjectCreateWithJSON(response)];
  [tableView reloadData];
  [label setCenter:[[theWindow contentView] center]];
}
{% endhighlight %}

The first method just submits the form when someone has selected a file, so that it uploads the file immediately when the user has pressed Open after selecting the file.
The second method changes the labels text and then adds the file to our array of files and redraws the table data so that it's shown.

Finally we'll take a quick look at FButtonView.j. (Again I had to remove the @ signs, be sure to note that).
{% highlight objc %}
import <Foundation/CPObject.j>

var DownloadIFrame = null,
    DownloadSlotNext = null;

implementation FButtonView : CPView
{
  [self super];
}

-(void)setObjectValue:anID
{
  //create download button and add as subview to second column
  downloadImage = [[CPImage alloc] initWithContentsOfFile:[[CPBundle mainBundle]
    pathForResource:@"download.png"]];
  [downloadImage setSize:CGSizeMake(20,20)];
  downloadButton = [[CPButton alloc] initWithFrame:CGRectMake( 0, 0, 20, 20 )];
  [downloadButton setBordered:NO];
  [downloadButton setImage:downloadImage];
  [downloadButton setAlternateImage:downloadImage];
  [downloadButton setImagePosition:CPImageOnly];
  [downloadButton setTarget:self];
  [downloadButton setTag:anID];
  [downloadButton setAction:@selector(downloader:)];
  [downloadButton sendActionOn:CPLeftMouseUpMask];
  [self addSubview:downloadButton];
}

- (void)downloader:(id)aButton
{
  if (DownloadIFrame == null)
  {
    DownloadIFrame = document.createElement("iframe");
    DownloadIFrame.style.position = "absolute";
    DownloadIFrame.style.top    = "-100px";
    DownloadIFrame.style.left   = "-100px";
    DownloadIFrame.style.height = "0px";
    DownloadIFrame.style.width  = "0px";
    document.body.appendChild(DownloadIFrame);
  }
  
  var now = new Date().getTime(),
      downloadSlot = (DownloadSlotNext && DownloadSlotNext > now)  ? DownloadSlotNext : now;
      
  DownloadSlotNext = downloadSlot + 2000;
  window.setTimeout(function() {
    if (DownloadIFrame != null)
      DownloadIFrame.src = "http://localhost:3000/uploads/"+[aButton tag];
    }, downloadSlot - now);
}

end
{% endhighlight %}

What happens when *tableView: objectValueForTableColumn: row:* gets called is that setObjectValue gets called on the DataView of the TableColumn with a parameter that is the object returned from objectValueForTableColumn.

I have implemented a subclass of CPView so that when setObjectValue gets called, a button is created that calls an action (downloader:) and the tag of the button is the ID of the file in the backend.

The downloader: method is taken from [this post](http://groups.google.com/group/objectivej/browse_thread/thread/e8242a8d91216741/5276536023f41678?lnk=gst&q=DownloadIFrame#5276536023f41678) in the [google group](http://groups.google.com/group/objectivej) and we don't really need to know how it works, but with some basic knowledge of HTML it's pretty obvious.

That's that! Now you know how to create a CPTableView, you know how to use the UploadButton and you know how to create a download button. I encourage you to download the code and play with it, it can be found here on github at [github.com/ique/tutorials](http://github.com/ique/tutorials).