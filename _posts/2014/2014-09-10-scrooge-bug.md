---
layout: post
title: "scrooge bug"
category: 
tags: []
---
{% include JB/setup %}


I use thrift 0.9.1
when I send a trace with one binary annotation to zipkin, it throws an exception:

>[error] WAR [20140910-18:26:29.693] processor: Invalid msg: CgABAEWSoa5p60QLAAMAAAAKZ2V0X21lbWJlcgoABABFkqGuaetEDwAGDAAAAAIKAAEABQKzfDHPowsAAgAAAAJjcwwAAwgAAQAAAAAGAAIAAAsAAwAAAAZrZXJuZWwAAAoAAQAFArN8Mc/jCwACAAAAAmNyAA8ACAwAAAABCwABAAAABGtleTESAAIAAAAGdmFsdWUxCAADAAAABgAA
[error] WAR [20140910-18:26:29.693] processor: org.apache.thrift.protocol.TProtocolException: Received wrong type for field 'value' (expected=STRING, actual=UNKNOWN).

I go to com.twitter.zipkin.gen.BinaryAnnotation.scala and print _field, and the output is as follows:
    
        [info] <TField name:'' type:18 field-id:2>

and it expect TType.STRING(namely type=11), but when I ignore the type mismatch like this(on BinaryAnnotation line 131):
        
        case 2 =>
            _field.`type` match {
              case TType.STRING => {
                value = readValueValue(_iprot)
              }
              case _actualType =>
                val _expectedType = TType.STRING
                value = readValueValue(_iprot)
                /*
                throw new TProtocolException(
                  "Received wrong type for field 'value' (expected=%s, actual=%s).".format(
                    ttypeToHuman(_expectedType),
                    ttypeToHuman(_actualType)
                  )
                )*/
            }
zipkin seems to accept it:
        
        [error] DEB [20140910-18:44:24.422] processor: Processing span:         
        Span(35613407032242887,get_member,           
       35613407032242887,None,List(Annotation(1410345864419960,cs,Some(0.0.0.0:0(kernel)),None), 
       Annotation(1410345864420008,cr,None,None)),ArrayBuffer(BinaryAnnotation(key1,java.nio.HeapByte
       Buffer[pos=0 lim=6 cap=6],AnnotationType(6,String),None)),false) from 
       CgABAH6GNHtl8scLAAMAAAAKZ2V0X21lbWJlcgoABAB
       +hjR7ZfLHDwAGDAAAAAIKAAEABQKzvEJaeAsAAgAAAAJjcwwAAwgAAQAAAAAGAAIAAAsAAwAAAAZrZXJuZWwAAAoAAQAFA
       rO8QlqoCwACAAAAAmNyAA8ACAwAAAABCwABAAAABGtleTESAAIAAAAGdmFsdWUxCAADAAAABgAA 

Is it a bug of scrooge?      