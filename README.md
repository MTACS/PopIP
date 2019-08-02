# PopIP
Display your device's IP address via an Activator gesture. Another tweak I decided to open source for beginners/developers wanting to use IP related stuff.

## Code

This tweak was a bit more complicated to make, I not only did I have to write an Activator listener, but I had to grab the device's IP using another source. I am sure that I could've grabbed it via copying it from Settings, but I found an easier way.

To set it up, I used the Activator listener NIC template from [here.](https://github.com/theos/templates "Official Theos templates") In terminal run /your/path/to/theos/bin/nic.pl or $THEOS/bin/nic.pl if you already exported $THEOS. I have an alias called tweak but it is the exact same thing. 

<p align="center">
  <img width="740" height="635" src="https://github.com/MTACS/MTACS.github.io/blob/master/images/popipnic.png">
</p>

Next cd to your tweak directory in some terminal. I use Hyper.is (check it out, much better than iTerm) and open it in your text editor, I use Atom (check it out, much better than Code or Sublime)

```bash
cd ~/Documents/Tweaks && atom .
```

Once it opens in Atom, we must change a few things to get the Activator listener working properly. Open Listener.xm and make the following changes: 

Before:

```objective-c
#include <objc/runtime.h>
#import <libactivator/libactivator.h>

@interface PopIPListener : NSObject <LAListener> {
	BOOL _isVisible;
	NSString *_bundleID;
}

+ (id)sharedInstance;

- (BOOL)present;
- (BOOL)dismiss;

@end

static id sharedActivatorIfExists(void) {
	static id *_LASharedActivator = nil;
	static dispatch_once_t token = 0;
```

Replace it with:

```objective-c
#include <objc/runtime.h>
#include <dlfcn.h>
#include <ifaddrs.h>
#include <arpa/inet.h>
#import <libactivator/libactivator.h>


@interface PopIPListener : NSObject <LAListener> {
	BOOL _isVisible;
	NSString *_bundleID;
}

+ (id)sharedInstance;

- (BOOL)present;
- (BOOL)dismiss;

@end

static LAActivator *sharedActivatorIfExists(void) {
	static LAActivator *_LASharedActivator = nil;
```

You can see we added a few #includes. We added <dlfcn.h> which is the header for supporting dynamic libraries. Activator uses the function dlopen(/usr/lib/libactivator.dylib) to enable Activator use in your tweak. Next we added <ifaddrs.h> and <arpa/inet.h> which are part of how we will grab our IP address. The last #import <libactivator/libactivator.h> is where our activator files are stored within Theos. Next change the id at the end of the last code snippet to LAActivator.

```objective-c
static id *sharedActivatorIfExists(void) { ---> static LAActivator *sharedActivatorIfExists(void) {
	static id *_LASharedActivator = nil; ---> static LAActivator *_LASharedActivator = nil;
```

Now our listener is all set up. Next we need to add the code to present the UIAlertViewController and add the IP address we will grab as the message.

I used a method found here to grab the address. It was set as a NSString but I simply removed the -(NSString)getIPAddress. I place the remaining code inside Activator's method to execute code once the assigned gesture is done. 

```objective-c
- (void)activator:(LAActivator *)activator receiveEvent:(LAEvent *)event forListenerName:(NSString *)listenerName {

	NSString *address = @"error";
	struct ifaddrs *interfaces = NULL;
	struct ifaddrs *temp_addr = NULL;
	int success = 0;

	success = getifaddrs(&interfaces);
	if (success == 0) {

			temp_addr = interfaces;
			while(temp_addr != NULL) {
					if(temp_addr->ifa_addr->sa_family == AF_INET) {
							// Check if interface is en0 which is the wifi connection on the iPhone
							if([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"]) {
									// Get NSString from C String
									address = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];

							}

					}

					temp_addr = temp_addr->ifa_next;
			}
	}

	freeifaddrs(interfaces);

	NSLog(@"%@", address);

	UIAlertController* alert = [UIAlertController alertControllerWithTitle:@"Current IP"
                           message:[NSString stringWithFormat:@"%@", address]
                           preferredStyle:UIAlertControllerStyleAlert];

													 UIAlertAction* defaultAction = [UIAlertAction actionWithTitle:@"Dismiss" style:UIAlertActionStyleDefault
                               handler:^(UIAlertAction * action) {}];

	[alert addAction:defaultAction];
	[[[[UIApplication sharedApplication] keyWindow] rootViewController] presentViewController:alert animated:YES completion:nil];

	if ([self presentOrDismiss]) {
		[event setHandled:YES];
	}
}
```

This uses C to grab the numerical IP address of en0, the main networking interface. Next I created a UIAlertController and set the Title to "Current UP", and the message to "message:[NSString stringWithFormat:@"%@", address]" with address being the device's IP. We must use stringWithFormat to convert it to a string, but use @"%@" to pass address later. Once compiled, PopIP can be set to any Activator gesture, anywhere because we added it to the current UIView with 

```objective-c
[[[[UIApplication sharedApplication] keyWindow] rootViewController] presentViewController:alert animated:YES completion:nil];
```

By doing this we grab the UIView that is being shown to the user, aka any view as it all uses UIView. Do your gesture and you should see the current IP of your device displayed as a UIAlert.

<p align="center">
  <img width="200" height="356" src="https://github.com/MTACS/MTACS.github.io/blob/master/images/popicalrt.jpg">
</p>
