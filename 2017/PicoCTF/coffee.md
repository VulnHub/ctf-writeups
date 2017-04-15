### Solved by superkojiman

We can decompile the class file using a Java decompiler: http://www.javadecompilers.com This gives us the following:

```
import java.util.Base64.Decoder;

public class problem {
  public problem() {}

  public static String get_flag() { String str1 = "Hint: Don't worry about the schematics";
    String str2 = "eux_Z]\\ayiqlog`s^hvnmwr[cpftbkjd";
    String str3 = "Zf91XhR7fa=ZVH2H=QlbvdHJx5omN2xc";
    byte[] arrayOfByte1 = str2.getBytes();
    byte[] arrayOfByte2 = str3.getBytes();
    byte[] arrayOfByte3 = new byte[arrayOfByte2.length];
    for (int i = 0; i < arrayOfByte2.length; i++)
    {
      arrayOfByte3[i] = arrayOfByte2[(arrayOfByte1[i] - 90)];
    }
    System.out.println(java.util.Arrays.toString(java.util.Base64.getDecoder().decode(arrayOfByte3)));
    return new String(java.util.Base64.getDecoder().decode(arrayOfByte3));
  }

  public static void main(String[] paramArrayOfString) {
    System.out.println("Nothing to see here");
  }
}
```

Just modify it so main() calls get_flag() and run it:

```
$ java problem
[102, 108, 97, 103, 95, 123, 112, 114, 101, 116, 116, 121, 95, 99, 111, 111, 108, 95, 104, 117, 104, 125]
```

Convert to ASCII and we get

```
flag_{pretty_cool_huh}
```
