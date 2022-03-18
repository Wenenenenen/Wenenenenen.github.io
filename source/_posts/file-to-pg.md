---
title: 大批量大文件导入pg数据库
index_img: /img/index_post/pg.jpg # https://s2.loli.net/2022/03/18/ZsMx2dSR96cK41f.png
date: 2022-03-18 10:03:25
tags: [PG, 数据库]
categories: 数据库
excerpt: 场景：读取800G的TXT文本文件到PG数据库，其中有单个超3G的文件，且格式并不完全相同，需要清洗
---
## 读大文件内存溢出

> 1. 读取大批量文件入库时的报错 `java.lang.OutOfMemoryError:GC overhead limit exceeded`
> 2. 最终发现是读单个大文件报错 `java.lang.OutOfMemoryError: Java heap space`
> 3. 每3000条插入一次  速度太慢  改端口再启动一个程序同时跑另一个文件夹  同时多次请求造成多线程同时执行

> 解决方法 -> 循环读文件时 每3000条提交一次后清空list 就不会有那么多`fp`对象积压在内存里了

```java
private String handlerOneFile(File file) {
    StringBuilder err = new StringBuilder();
    try {
      DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
      BufferedInputStream fis = new BufferedInputStream(new FileInputStream(file));
      // 用50M的缓冲读取文本文件
      BufferedReader reader = new BufferedReader(
              new InputStreamReader(fis, StandardCharsets.UTF_8),50*1024*1024);

      String row;
      List<CarLineFp> fpList = new ArrayList<>();
      int num = 0;
      while((row = reader.readLine()) != null){
        String[] split = row.split(",");
        CarLineFp fp = new CarLineFp();
        fp.setCarPlace(split[0].substring(0, 1));
        fp.setCarId(split[0].substring(1));
        fp.setPositionTime(LocalDateTime.parse(split[1], formatter));
        fp.setLongitude(split[2]);
        fp.setLatitude(split[3]);
        fp.setCarSpeed(split[4]);
        fp.setIndCode(split[5]);
        fp.setDirectionAngle(split[6]);
        fp.setCarStateSign(split[7]);
        fp.setRegionCode(split[8]);

        fpList.add(fp);
        num++;
        System.out.println("计数 ---> "+ num);
        // 3000条时候先提交一次
        if(fpList.size() == 3000){
          System.out.println("*******进入提交3000条");
          addBatch(fpList);
          System.out.println("*******提交3000条成功");
          fpList.clear();
        }
      }
      // 提交3000条的余数
      addBatch(fpList);
      log.info("*****************************"+file.getName() +" 行数 --> "+ num);
      reader.close();
    }catch (Exception e) {
      log.error(file.getName()+" 报错 --> "+e.getMessage());
      err.append(file.getName()).append(" 报错 --> ").append(e.getMessage());
    }

    return err.toString();
  }
```

> 再次更换思路优化  使用copy命令  效率显著提高  不过要先过滤一次重叠行的数据(加换行符或者删除)

`copy wjy.car_line_fp_for_copy from 'E:\Desktop\TestProject\car\m08d07\new.txt' with csv`

> 由于文件和数据库不在同一个服务器上 直接from文件行不通 需要通过文件流的方式，并且使用`copyManager`
>
> 不过需要自己新建connect，不能用`mybatis`了

```java
private void addWithIO(Connection connection, String full) throws Exception{
  FileInputStream fileInputStream = null;
  String sql = "COPY yanjiu.car_line_fp FROM STDIN WITH CSV";
  try {
    CopyManager copyManager = new CopyManager((BaseConnection) connection);
    // full 是完整文件路径
    fileInputStream = new FileInputStream(full);
    copyManager.copyIn(sql, fileInputStream);
  } finally {
    if(fileInputStream != null) {
      try {
        fileInputStream.close();
      }catch (Exception e) {
        e.printStackTrace();
      }
    }
  }
}
```
