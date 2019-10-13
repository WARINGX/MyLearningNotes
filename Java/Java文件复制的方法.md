### 1. 使用FileStreams复制

这是最经典的方式将一个文件的内容复制到另一个文件中。 使用FileInputStream读取文件source的字节，使用FileOutputStream写入到文件dest。 

### 2. 使用FileChannel复制

Java NIO包括transferFrom方法,根据文档应该比文件流复制的速度更快。 

### 3. 使用Java7的Files类复制

如果你有一些经验在Java 7中你可能会知道,可以使用复制方法的Files类文件,从一个文件复制到另一个文件。 

### 4. 使用Commons IO复制

Apache Commons IO提供拷贝文件方法在其FileUtils类,可用于复制一个文件到另一个地方。

代码示例：

```
package test;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.channels.FileChannel;
import java.nio.file.Files;
import org.apache.commons.io.FileUtils;
public class test {
    
    public static void main(String[] args) throws Exception{
    	File source=new File("D:\\MyLearningNotes\\test.txt");
    	File dest1=new File("D:\\MyLearningNotes\\test1.txt");
    	File dest2=new File("D:\\MyLearningNotes\\test2.txt");
    	File dest3=new File("D:\\MyLearningNotes\\test3.txt");
    	File dest4=new File("D:\\MyLearningNotes\\test4.txt");
    	copyFileUsingFileStreams( source, dest1);
    	copyFileUsingFileChannels(source, dest2);
    	copyFileUsingJava7Files(source, dest3);
    	copyFileUsingApacheCommonsIO(source, dest4);
    }
//1.Java流和File类
	private static void copyFileUsingFileStreams(File source, File dest)
	        throws IOException {    
	    InputStream input = null;    
	    OutputStream output = null;    
	    try {
	           input = new FileInputStream(source);
	           output = new FileOutputStream(dest);        
	           byte[] buf = new byte[1024];        
	           int bytesRead;        
	           while ((bytesRead = input.read(buf)) > 0) {
	               output.write(buf, 0, bytesRead);
	           }
	    } finally {
	        input.close();
	        output.close();
	    }
	}
//2.java.nio.channels.FileChannel类
	private static void copyFileUsingFileChannels(File source, File dest) throws 					IOException {    
        FileChannel inputChannel = null;    
        FileChannel outputChannel = null;    
	    try {
	        inputChannel = new FileInputStream(source).getChannel();
	        outputChannel = new FileOutputStream(dest).getChannel();
	        outputChannel.transferFrom(inputChannel, 0, inputChannel.size());
	    } finally {
	        inputChannel.close();
	        outputChannel.close();
	    }
    
	}
//3.Java7 java.nio.file.Files类
	private static void copyFileUsingJava7Files(File source, File dest)
	        throws IOException {    
	        Files.copy(source.toPath(), dest.toPath());
	}
//4.org.apache.commons.io.FileUtils类
	private static void copyFileUsingApacheCommonsIO(File source, File dest)
	        throws IOException {
	    FileUtils.copyFile(source, dest);
	}
}
```

