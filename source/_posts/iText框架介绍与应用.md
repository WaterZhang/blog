---
title: iText框架介绍与应用
date: 2016-11-16 09:50:13
tags:
- Java
- iText
categories: 工作总结
toc: true
---

在工作中，需要结合html和image以及动态的信息（Business Model）生成一些静态信息，比如widget，图片，PDF。在google中找到**[iText](http://itextpdf.com/)**框架，帮忙解决html转换为PDF的方案。

## 版本选择
iText提供[iText 5](http://developers.itextpdf.com/examples-itext5)和[iText 7](http://developers.itextpdf.com/examples-itext7)，两个不同版本。刚开始直接使用最新的iText7，在测试demo时，发现iText5的例子多很多，而且XmlWorker不支持iText7，则转向了更成熟的iText 5。
Maven，
~~~java
<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>itextpdf</artifactId>
  <version>5.5.10</version>
</dependency>

<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>itext-pdfa</artifactId>
  <version>5.5.10</version>
</dependency>

<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>itext-xtra</artifactId>
  <version>5.5.10</version>
</dependency>

<dependency>
  <groupId>com.itextpdf.tool</groupId>
  <artifactId>xmlworker</artifactId>
  <version>5.5.10</version>
</dependency>
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itextg</artifactId>
    <version>5.5.10</version>
</dependency>
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itext-asian</artifactId>
    <version>5.2.0</version>
</dependency>
~~~

Gradle，
~~~java
compile "com.itextpdf:itextpdf:$itextVersion"
compile "com.itextpdf:itext-pdfa:$itextVersion"
compile "com.itextpdf:itext-xtra:$itextVersion"
compile "com.itextpdf:itextg:$itextVersion"
compile "com.itextpdf.tool:xmlworker:$itextVersion"
compile 'com.itextpdf:itext-asian:5.2.0'
~~~

## Html转PDF
刚开始测试时，直接使用io引入html源码，iText框架提供XmlWorker帮忙转成PDF，多方便，如果能这么简单多好。
~~~java
public void createPdf(String file) throws IOException, DocumentException {
    // step 1
    Document document = new Document();
    // step 2
    PdfWriter writer = PdfWriter.getInstance(document, new FileOutputStream(file));
    writer.setInitialLeading(12);
    // step 3
    document.open();
    // step 4
    XMLWorkerHelper.getInstance().parseXHtml(writer, document,
            new FileInputStream(HTML));
    // step 5
    document.close();
}
~~~
但是真正工作上使用，这方案是，行不通，复杂的CSS解析不了，后续不得不寻找其他方法。在官网上没有找到支持哪些CSS功能，哪些不支持。自己测试发现，CSS的selector过于复杂不支持。

## Image方案
工作上的需求，是将照片和业务信息结合展示。那么看看iText是否对image支持。发现有[image examples](http://developers.itextpdf.com/examples/image-examples-itext5)。便将方案调整为Image+Watermark（照片+水印）的方案，绕过HTML的方法。剩下的问题，就是调整文本内容的位置，大小，颜色和字体了。
样例代码，
~~~java
public static final String IMG = "resources/images/test.png";
public static final String DEST = "results/test.pdf";
public static final String HOTEL_SANST = "resources/fonts/test.otf";
public static final String HOTEL_SANS_LIGHT = "resources/fonts/test Light.otf";

public static final Font FONT = new Font(FontFamily.HELVETICA, 15, Font.NORMAL, BaseColor.RED);

public static void main(String[] args) throws Exception {
    Image img = Image.getInstance(IMG);
    Document document = new Document(PageSize.A4);
    PdfWriter writer = PdfWriter.getInstance(document, new FileOutputStream(DEST));
    document.open();

    img.setAbsolutePosition(0, 0);
    writer.setStrictImageSequence(true);
    PdfContentByte cb = writer.getDirectContentUnder();
    Image image =getWatermarkedImage(cb,document.getPageSize(),img, "Bruno");
    image.setAbsolutePosition(0, 0);
    document.add(image);

    document.close();

}


public static Image getWatermarkedImage(PdfContentByte cb,Rectangle position, Image img, String watermark) throws DocumentException {


    BaseFont hotel_font = null;
    BaseFont hotel_font_light = null;
    try {
        hotel_font = BaseFont.createFont(HOTEL_SANST, BaseFont.IDENTITY_H, BaseFont.EMBEDDED);
        hotel_font_light = BaseFont.createFont(HOTEL_SANS_LIGHT, BaseFont.IDENTITY_H, BaseFont.EMBEDDED);

    } catch (DocumentException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    } catch (IOException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }


    PdfTemplate template = cb.createTemplate(position.getWidth(), position.getHeight());
    template.addImage(img);

    ColumnText.showTextAligned(template, Element.ALIGN_CENTER,
            new Phrase("2016", new Font(hotel_font, 40, Font.NORMAL, BaseColor.RED)),
            (position.getLeft() + position.getRight()) / 2,
            position.getTop() - 60, 0);
        ColumnText.showTextAligned(template, Element.ALIGN_CENTER,
                new Phrase("Test name", new Font(hotel_font, 25, Font.NORMAL, BaseColor.RED)),
                (position.getLeft() + position.getRight()) / 2,
                position.getTop() - 150, 0);
        ColumnText.showTextAligned(template, Element.ALIGN_CENTER,
                new Phrase("Haha", new Font(hotel_font_light, 130, Font.NORMAL, BaseColor.RED)),
                (position.getLeft() + position.getRight()) / 2,
                (position.getBottom() + position.getTop()) / 2 + 30, 0);
        ColumnText.showTextAligned(template, Element.ALIGN_CENTER,
                new Phrase("3.6", new Font(hotel_font, 130, Font.NORMAL, BaseColor.RED)),
                (position.getLeft() + position.getRight()) / 2 ,
                (position.getBottom() + position.getTop()) / 2 - 100, 0);

        return Image.getInstance(template);
}
~~~

## 后续
还有一个难题，是多语言问题，在测试demo时，发现亚洲语言的支持不是很好，有些内容不会在PDF中显示出来。到时候要转换成Unicode码试试。