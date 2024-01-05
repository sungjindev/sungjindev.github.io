---
title: 순수 Java 코드로 Excel data 읽어서 파싱하는 법과 주의점
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, excel parsing, Apache POI, 백엔드, 스프링, 자바, 엑셀 파싱, 아파치 포이]
---

이번 포스팅에서는 Spring으로 사이드 프로젝트를 진행하며 Excel dataset에 대한 Parsing이 필요했었는데 그때 Apache POI를 사용하면서 겪었던 문제점들과 함께 간단한 사용 방법을 소개하려고 합니다.

## Apache POI란?
Apache POI는 아파치 소프트웨어 재단에서 만든 라이브러리로 마이크로소프트 오피스 파일들을 순수 자바 코드로 읽고 쓸 수 있게 해주는 기능을 제공합니다.   
    
POI라는 이름은 "Poor Obfuscation Implementation"의 줄임말인데 여기에는 제법 재밌는 유래가 있습니다. 기존의 마이크로소프트 오피스의 파일 포맷이 일부러 해독하기 어렵게 만든 것이 아니냐라고 생각될 정도로 불편했지만, 그럼에도 불구하고 리버스 엔지니어링하여 비로소 Apache POI가 탄생할 수 있게되었다라는 의미를 담고 있습니다. 그래서 그런지 POI 프로젝트 내에 많은 모듈들이 존재하는데 이들의 이름도 이와 비슷하게 유머섞인 이름이 많다고 합니다.   
    
## Apache POI로 Excel file 읽기
코드를 보면 쉽게 이해할 수 있어서 코드부터 살펴보겠습니다.
```java
public static List<StoreDataByParser> readExcelData() {
        List<StoreDataByParser> storeDataByParserList = new ArrayList<>();

        //프로젝트 폴더 내 resources/data 폴더에 접근하기 위해 절대 경로를 설정
        //Windows와 Linux의 경로 표기법이 \와 /로 다르므로 이를 모두 호환하기 위해 File.separator 사용
        String absolutePath = new File("").getAbsolutePath() + File.separator + "src" + File.separator + "main" + File.separator + "resources" + File.separator;
        String fileName = "fileName.xlsx";   //데이터를 추출한 데이터셋 Excel 파일명

        String filePath = absolutePath + fileName;
        String extension = FilenameUtils.getExtension(fileName);

        try (FileInputStream fileInputStream = new FileInputStream(new File(filePath))) {
            Workbook workbook = null;
            if (extension.equals("xlsx")) { //파일 확장자를 읽어들여서 xlsx 파일이면 XSSFWorkBook 포맷으로 생성
                workbook = new XSSFWorkbook(fileInputStream);
            } else if (extension.equals("xls")) {   //xls 파일이면 HSSFWorkBook 포맷으로 생성
                workbook = new HSSFWorkbook(fileInputStream);
            } else {
                throw new IOException("올바르지 않은 파일 확장자입니다. Excel 파일 확장자만 허용됩니다.");
            }

            Sheet sheet = workbook.getSheetAt(0);   //데이터를 읽어들일 Excel 파일의 sheet number

            for (int i = 1; i < sheet.getLastRowNum(); i++) {   //0번째 Row는 column title이라 제외하고 1번부터 마지막 Row까지 조회
                Row row = sheet.getRow(i);

                Cell statusCell = row.getCell(0);
                if (isCellEmpty(statusCell)) {  //이렇게 isCellEmpty() Method처럼 다중으로 검증해주지 않으면 POI에서 제대로 Blank cell이나 Empty cell에 대해서 처리를 잘 못한다.
                    continue;
                }

                if (statusCell.getCellType() == CellType.STRING) {  //빡빡하게 타입 검사! String 일때만 비교하기
                    if(statusCell.getStringCellValue().equals("폐업"))    //영업 상태가 폐업이면 skip
                        continue;
                }

                StoreDataByParser storeDataByParser = new StoreDataByParser();

                Cell nameCell = row.getCell(1);
                if(isCellEmpty(nameCell))   //Empty cell, Blank cell Empty string 여부를 검사
                    continue;

                if (nameCell.getCellType() == CellType.STRING) {
                    storeDataByParser.setName(nameCell.getStringCellValue());
                }

                storeDataByParserList.add(storeDataByParser);   //위에서 파싱한 각 가게별 데이터를 List에 담기
            }
        } catch (IOException e) {   //입출력 예외처리
            e.printStackTrace();
        }
        
        return storeDataByParserList;
    }

    public static boolean isCellEmpty(final Cell cell) {    //POI에서 Empty cell이나 Blank cell을 처리하는 로직이 까다로워서 따로 메서드로 추출
        if (cell == null) {
            return true;
        } else if (cell.getCellType() == CellType.BLANK) {
            return true;
        } else if (cell.getCellType() == CellType.STRING && cell.getStringCellValue().trim().isEmpty()) {
            return true;
        }

        return false;
    }
```

코드에 대한 자세한 설명은 모든 Line에 주석을 달아놔서 전반적인 흐름은 쉽게 이해할 수 있을 것이라 생각합니다. 간단히 살펴보면 위 코드는 가게 데이터들을 담은 Excel 파일을 읽어서 필요한 데이터를 파싱하는 작업을 담고있습니다. 여기서 가장 중요한 부분이라하면 당연 isCellEmpty() 메서드입니다.   
    
사실 이 메서드를 구현하기 전에는 굉장히 큰 오류에 시달렸는데요. Excel에는 Empty cell, Blank cell이라는 것이 존재합니다. 이게 무엇을 의미하냐면, Cell data를 육안으로 봤을 때 모두 똑같은 빈칸으로 보이지만 그 Cell이 실제로 null일 수도 있고 아니면 null이 아닌 ""와 같은 empty string일 수도 있습니다.   
   
저도 Apache POI를 처음 써보면서 이 사실을 잘모르고 예외처리를 제대로하지 않았다가 계속 의도한대로 동작하지 않는 오류에 빠졌습니다. 이게 찾아보다보니, 이런 원인과 이유에 대해서 자세히 설명하고 있는 블로그가 많이 없어서 트러블 슈팅하는데 더욱 애를 먹었던 것 같습니다.   
    
그래서, 뭔가 Apache POI를 통해 데이터를 읽어오는데 의도치않게 데이터가 잘 걸러지지 않는다던지 null로 나온다던지하면 위 케이스를 꼭 검사해보시길 바랍니다.   

## 마무리
이번 포스팅에서는 Apache POI를 사용해서 순수 Java 코드로 Excel dataset들을 읽어들이고 파싱하는 방법에 대해 알아봤습니다. 코드 자체가 이해하기는 어렵지 않은데, Empty cell이나 Blank cell 등에 대해서 제대로 예외처리를 해주지 않으면 오류에 대한 인과를 알기가 굉장히 까다로워집니다. 저와같은 이유로 소중한 시간을 허비하지 않으셨으면 하는 마음에 이 글로 공유드립니다.