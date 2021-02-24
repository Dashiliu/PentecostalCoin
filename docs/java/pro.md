
java 读取大容量文件，内存溢出？怎么按几行读取，读取多次

 import java.io.BufferedReader;

import java.io.FileNotFoundException;

import java.io.FileReader;

import java.io.IOException;

import java.io.RandomAccessFile;

import java.util.Scanner;

 

public class TestPrint {

public static void main(String[] args) throws IOException {

String path = "你要读的文件的路径";

RandomAccessFile br=new RandomAccessFile(path,"rw");//这里rw看你了。要是之都就只写r

String str = null, app = null;

int i=0;

while ((str = br.readLine()) != null) {

​	i++;

​	app=app+str;

​	if(i>=100){//假设读取100行

​		i=0;

​		// 这里你先对这100行操作，然后继续读

​		app=null;

   }

}

br.close();

} 

}

