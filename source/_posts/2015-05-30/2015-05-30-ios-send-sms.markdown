---
layout: post
title: "ios send SMS"
date: 2015-05-30 22:27:02 +0800
comments: true
categories: IOS
---
```obj-C
#import <Foundation/Foundation.h>
#import <termios.h>
#import <time.h>
#import <sys/ioctl.h>
 
@implementation NSString(UCS2Encoding)

- (NSString*)ucs2EncodingString{
    NSMutableString *result = [NSMutableString string];
    for (int i = 0; i < [self length]; i++) {
        unichar unic = [self characterAtIndex:i];
        [result appendFormat:@"%04hX",unic];
    }
    return [NSString stringWithString:result];
}
 
- (NSString*)ucs2DecodingString{
    NSUInteger length = [self length]/4;
    unichar *buf = malloc(sizeof(unichar)*length);
    const char *scanString = [self UTF8String];
    for (int i = 0; i < length; i++) {
        sscanf(scanString+i*4, "%04hX", buf+i);
    }
    return [[NSString alloc] initWithCharacters:buf length:length];
}

- (NSString*)hexSwipString{
    unichar *oldBuf = malloc([self length]*sizeof(unichar));
    unichar *newBuf = malloc([self length]*sizeof(unichar));
    [self getCharacters:oldBuf range:NSMakeRange(0, [self length])];
    for (int i = 0; i < [self length]; i+=2) {
        newBuf[i] = oldBuf[i+1];
        newBuf[i+1] = oldBuf[i];
    }
    NSString *result = [NSString stringWithCharacters:newBuf length:[self length]];
    free(oldBuf);
    free(newBuf);
    return result;
}
 
@end
```

<!--more-->

```
NSString *PDUEncodeSendingSMS(NSString *phone, NSString *text){
    NSMutableString *string = [NSMutableString stringWithString:@"001100"];
    [string appendFormat:@"%02X", (int)[phone length]];
    if ([phone length]%2 != 0) {
        phone = [phone stringByAppendingString:@"F"];
    }
    [string appendFormat:@"81%@",[phone hexSwipString]];
    [string appendString:@"0008AA"];
    NSString *ucs2Text = [text ucs2EncodingString];
    [string appendFormat:@"%02x%@", (int)[ucs2Text length]/2, ucs2Text];
    return [NSString stringWithString:string];
}
 
NSString *sendATCommand(NSFileHandle *baseband, NSString *atCommand){
    NSLog(@"SEND AT: %@", atCommand);
    [baseband writeData:[atCommand dataUsingEncoding:NSASCIIStringEncoding]];
    NSMutableString *result = [NSMutableString string];
    NSData *resultData = [baseband availableData];
    while ([resultData length]) {
        [result appendString:[[NSString alloc] initWithData:resultData encoding:NSASCIIStringEncoding]];
        if ([result hasSuffix:@"OK\r\n"]||[result hasSuffix:@"ERROR\r\n"]||[result rangeOfString:@">"].location != NSNotFound) {
            NSLog(@"RESULT: %@", result);
            return [NSString stringWithString:result];
        }
        else{
            resultData = [baseband availableData];
        }
    }
    return nil;
}
 
BOOL addNewSIMContact(NSFileHandle *baseband, NSString *name, NSString *phone){
    NSString *result = sendATCommand(baseband, [NSString stringWithFormat:@"AT+CPBW=,\"%@\",,\"%@\"\r", phone, [name ucs2EncodingString]]);
    if ([result hasSuffix:@"OK\r\n"]) {
        return YES;
    }
    else{
        return NO;
    }
}
 
BOOL sendSMSWithPDUMode(NSFileHandle *baseband, NSString *phone, NSString *text){
    NSString *pduString = PDUEncodeSendingSMS(phone, text);
    NSString *result = sendATCommand(baseband, [NSString stringWithFormat:@"AT+CMGS=%d\r", (int)[pduString length]/2-1]);
    result = sendATCommand(baseband, [NSString stringWithFormat:@"%@\x1A", pduString]);
    if ([result hasSuffix:@"OK\r\n"]) {
        return YES;
    }
    else{
        return NO;
    }
}
 
NSArray *readAllSIMContacts(NSFileHandle *baseband){
    NSString *result = sendATCommand(baseband, @"AT+CPBR=?\r");
    if (![result hasSuffix:@"OK\r\n"]) {
        return nil;
    }
    int max = 0;
    sscanf([result UTF8String], "%*[^+]+CPBR: (%*d-%d)", &max);
    result = sendATCommand(baseband, [NSString stringWithFormat:@"AT+CPBR=1,%d\r",max]);
    NSMutableArray *records = [NSMutableArray array];
    NSScanner *scanner = [NSScanner scannerWithString:result];
    [scanner scanUpToString:@"+CPBR:" intoString:NULL];
    while ([scanner scanString:@"+CPBR:" intoString:NULL]) {
        NSString *phone = nil;
        NSString *name = nil;
        [scanner scanInt:NULL];
        [scanner scanString:@",\"" intoString:NULL];
        [scanner scanUpToString:@"\"" intoString:&phone];
        [scanner scanString:@"\"," intoString:NULL];
        [scanner scanInt:NULL];
        [scanner scanString:@",\"" intoString:NULL];
        [scanner scanUpToString:@"\"" intoString:&name];
        [scanner scanUpToString:@"+CPBR:" intoString:NULL];
        if ([phone length] > 0 && [name length] > 0) {
            [records addObject:@{@"name":[name ucs2DecodingString], @"phone":phone}];
        }
    }
    return [NSArray arrayWithArray:records];
}
```


```
int main(int argc, const char * argv[])
{
    
    @autoreleasepool {
        
        NSFileHandle *baseband = [NSFileHandle fileHandleForUpdatingAtPath:@"/dev/dlci.spi-baseband.extra_0"];
        if (baseband == nil) {
            NSLog(@"Can't open baseband.");
        }
        
        int fd = [baseband fileDescriptor];
        
        ioctl(fd, TIOCEXCL);
        fcntl(fd, F_SETFL, 0);
        
        static struct termios term;
        
        tcgetattr(fd, &term);
        
        cfmakeraw(&term);
        cfsetspeed(&term, 115200);
        term.c_cflag = CS8 | CLOCAL | CREAD;
        term.c_iflag = 0;
        term.c_oflag = 0;
        term.c_lflag = 0;
        term.c_cc[VMIN] = 0;
        term.c_cc[VTIME] = 0;
        tcsetattr(fd, TCSANOW, &term);
        
        
        NSString *result = sendATCommand(baseband, @"AT+CSCS=\"UCS2\"\r");
        result = sendATCommand(baseband, @"ATE0\r");
        result = sendATCommand(baseband, @"AT+CMGF=0\r");
 
        
        sendSMSWithPDUMode(baseband, @"10010", @"测试");
        
 
    }
    return 0;
}
```
