To build it simply do: buildh.bat tutor01

```
static bError, s_cTableName, s_cEngine, s_cServer, s_cUserfName, s_cPassword, s_cQuery := ""
 
// Valores temporales hasta que hagamos una conexion real
// Temporary values until we do a real connection
static s_aStruct
 
#include "rddsys.ch"
#include "fileio.ch"
#include "error.ch"
#include "common.ch"
#include "dbstruct.ch"
 
#ifndef __XHARBOUR__
   #include "hbusrrdd.ch"
   #xcommand TRY              => bError := errorBlock( {|oErr| break( oErr ) } ) ;;
                                 BEGIN SEQUENCE
   #xcommand CATCH [<!oErr!>] => errorBlock( bError ) ;;
                                 RECOVER [USING <oErr>] <-oErr-> ;;
                                 errorBlock( bError )
#else
   #include "usrrdd.ch"
   #define HB_SYMBOL_UNUSED( symbol )  ( symbol := ( symbol ) )
#endif
 
#define WA_RECORDSET           1
#define WA_BOF                 2
#define WA_EOF                 3
#define WA_CONNECTION          4
#define WA_CATALOG             5
#define WA_TABLENAME           6
#define WA_ENGINE              7
#define WA_SERVER              8
#define WA_USERNAME            9
#define WA_PASSWORD           10
#define WA_QUERY              11
#define WA_LOCATEFOR          12
#define WA_SCOPEINFO          13
#define WA_SQLSTRUCT          14
#define WA_FCOUNT             15
#define WA_RECNO              16 // temporary values meanwhile we don't do a real connection
#define WA_RECCOUNT           17 // temporary values meanwhile we don't do a real connection
#define WA_APPEND             18 // dbAppend() flag
#define WA_RECORDCHANGED      19 // pArea->fRecordChanged flag
#define WA_INVALIDBUFFER      20 // Indicates if current buffer is invalid 
#define WA_TRANSCOUNTER       21 // An counter for the transactions currently open.
#define WA_SYSTEMID           22 // It indicates the driver in use
 
/* For buffer operations with GET/PUT values */
#define WA_BUFFER_ARR         23 // Buffer with changed values. Values unchanged will be recorded as NIL on this array. (O correto seria ter 1 segundo array pra controlar isto)
#define WA_BUFFER_POS         24 // Buffer with current row number
#define WA_BUFFER_ROWCOUNT    25 // Number of lines in buffer to avoid LEN(WA_BUFFER_ARR) 
 
#define WA_SIZE               25
 
 
// list the drivers currently supported
#define ID_NONE                0
#define ID_MYSQL               1
#define ID_POSTGRESQL          2
#define ID_ACCESS              3
 
// Common control fields
#define FLD_RECNO      "sql_recno"
 
ANNOUNCE SQLWIN
 
// static bError, s_cTableName, s_cEngine, s_cServer, s_cUserName, s_cPassword, s_cQuery := ""

function Main()
 
   ErrorBlock( { | oError | MsgInfo( ErrorMessage( oError ) + CRLF + ;
                                     Str( ProcLine( 2 ) ) , "Error" ), __Quit() } ) 
 
   /*
    * Try to change the SQLWIN.PRG on line 107 to work with POSTGRESQL and recompile
    * this test, please.
    */
   DbCreate( "test.dbf", { { "first", "C", 10, 0 },;
                           { "last",  "C", 10, 0 },;
                           { "age",   "N",  3, 0 },;
                           { "data",  "D", 10, 0 } }, "SQLWIN" )
 
   USE test VIA "SQLWIN"  
 
   MsgInfo( "Alias: " + Alias() ) 
   MsgInfo( test->( FieldName( 1 ) ) )                     
 
   GOTO 10
 
   test->first = "Paul"
   FieldPut( 2, "Smith" )
   FieldPut( FieldPos( "data" ), DATE() )
   COMMIT
 
   GOTO BOTTOM
 
   MsgInfo( RecNo(), 'Recno()' )
   MsgInfo( RecCount(), 'RecCount()' )
 
  /*
   * Note que SET DELETED affect theses functions
   */
   SET DELETED ON
   MsgInfo( RecNo(), 'Recno()' )
   MsgInfo( RecCount(), 'RecCount()' )
 
   APPEND BLANK
   test->first = "Paul's"
   FieldPut( 2, "Smith" )
   FieldPut( FieldPos( "data" ), CTOD("") )
   COMMIT
 
   MsgInfo( test->First, "test->First" )
   MsgInfo( RecCount(), 'RecCount()' )
 
   PACK                                               
   ZAP
 
return nil        

static func ErrorMessage( e )

    local cMessage := if( empty( e:OsCode ), ;
                          if( e:severity > ES_WARNING, "Error ", "Warning " ),;
                          "(DOS Error " + AllTrim( Str( e:osCode ) ) + ") " )

    cMessage += if( ValType( e:SubSystem ) == "C",;
                    e:SubSystem()                ,;
                    "???" )

    cMessage += if( ValType( e:SubCode ) == "N",;
                    "/" + AllTrim( Str( e:SubCode ) ) ,;
                    "/???" )
  if ( ValType( e:Description ) == "C" )
        cMessage += "  " + e:Description
   end

    cMessage += if( ! Empty( e:FileName ),;
                    ": " + e:FileName   ,;
                    if( !Empty( e:Operation ),;
                        ": " + e:Operation   ,;
                        "" ) )
return cMessage
 
#ifdef __XHARBOUR__
 
static function HB_TokenGet( cText, nPos, cSep )
 
   local aTokens := HB_ATokens( cText, cSep )
 
return If( nPos <= Len( aTokens ), aTokens[ nPos ], "" )
 
#endif
 
static function SQL_INIT( nRDD )
 
   local aRData
 
   USRRDD_RDDDATA( nRDD, aRData )
 
return SUCCESS
 
static function SQL_NEW( nWA )
 
   local aWAData := Array( WA_SIZE )
 
   aWAData[ WA_BOF ] := .F.
   aWAData[ WA_EOF ] := .F.
   aWAData[ WA_RECCOUNT ] := 0
 
   aWAData[ WA_BUFFER_ARR ]      := NIL
   aWAData[ WA_BUFFER_POS ]      := 0
   aWAData[ WA_BUFFER_ROWCOUNT ] := 0
   aWAData[ WA_INVALIDBUFFER ]   := .F.
   aWAData[ WA_APPEND ]          := .F.
   aWAData[ WA_RECORDCHANGED ]   := .F.
 
   aWAData[ WA_SYSTEMID ]    := ID_MYSQL // ID_POSTGRESQL // ID_NONE
   aWAData[ WA_TRANSCOUNTER] := 0
 
   USRRDD_AREADATA( nWA, aWAData )
 
return SUCCESS
 
static function SQL_CREATE( nWA, aOpenInfo )
 
   local cDataBase   := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 1, ";" )  // This is a SQL Table name ??? - Vailton
   local cTableName  := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 2, ";" )
   local cDbEngine   := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 3, ";" )
   local cServer     := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 4, ";" )
   local cUserName   := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 5, ";" )
   local cPassword   := HB_TokenGet( aOpenInfo[ UR_OI_NAME ], 6, ";" )
   local aWAData     := USRRDD_AREADATA( nWA )
   local oError
   local cSQL
   local cSchema, cSequenceField
 
   do case
      case Lower( Right( cDataBase, 4 ) ) == ".mdb"
           if ! File( cDataBase )
              // oCatalog:Create( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + cDataBase )
           endif
           // oConnection:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + cDataBase )
 
           aWAData[ WA_SYSTEMID ] := ID_ACCESS
 
      case Upper( cDbEngine ) == "MYSQL"
           /*
           oConnection:Open( "DRIVER={MySQL ODBC 3.51 Driver};" + ;
                             "server=" + cServer + ;
                             ";database=" + cDataBase + ;
                             ";uid=" + cUserName + ;
                             ";pwd=" + cPassword )
                             */
           aWAData[ WA_SYSTEMID ] := ID_MYSQL
 
   endcase
 
   cDataBase := SQLAdjustFn( cDataBase )
 
   TRY
      DO CASE
      CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL
         cSQL = "DROP TABLE `" + cDataBase + "` IF EXIST" + Chr( 13 ) + Chr( 10 )
 
      CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
         IF ValType( cSchema ) == 'U'
            cSchema := 'public'
         End
         * We will send the DROP command below ...  
      End
   CATCH
   END
 
   TRY
      DO CASE
      CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL
         cSQL += "CREATE TABLE `" + cDataBase + "` ( " + aWAData[ WA_SQLSTRUCT ] + " )"
 
      CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
 
         /*
          * In some drivers - such as PostgreSQL - are needed more than one command
          * to create the table. This should be seen  within the routine that build 
          * the SQL strings. (Vailton)
          * 15/09/2008 - 20:08:47
          */
         cSequenceField := cDataBase + "_" + FLD_RECNO
 
         /* We destroy the sequence to create the table and ignore any error */
         cSQL := 'DROP SEQUENCE "'+cSchema + '"."' + cSequenceField +'"'
         MsgInfo( cSQL + " // DbCreate()" )
         cSQL := ''         
 
         /* We created the sequence to create the table */
         cSQL := 'CREATE SEQUENCE "' + cSchema + '"."' + cSequenceField + '"'
         cSQL += ' INCREMENT 1  MINVALUE 1'
         cSQL += ' CACHE 1;'
 
         MsgInfo( cSQL + " // DbCreate()" )
         cSQL := ''         
 
         cSQL += 'DROP TABLE "' + cSchema + '"."' + cDataBase + '"'
         MsgInfo( cSQL + " // DbCreate()" )
         cSQL := ''         
 
         /* We created the table now */
         cSQL := 'CREATE TABLE "' + cSchema + '"."' + cDataBase + '" (' +;
                                     aWAData[ WA_SQLSTRUCT ]
 
         cSQL += ", " + FLD_RECNO + " NUMERIC(15,0) DEFAULT nextval('" +;
                                           cSequenceField + "'::regclass) UNIQUE NOT NULL"
         cSQL += ')'
      End
 
      MsgInfo( cSQL + " // DbCreate()" )
      cSQL := ''         
   CATCH
      oError := ErrorNew()
      oError:GenCode     := EG_CREATE
      oError:SubCode     := 1004
      oError:Description := HB_LANGERRMSG( EG_CREATE ) + " (" + ;
                            HB_LANGERRMSG( EG_UNSUPPORTED ) + ")"
      oError:FileName    := aOpenInfo[ UR_OI_NAME ]
      oError:CanDefault  := .T.
      UR_SUPER_ERROR( nWA, oError )
   END
   // oConnection:Close()
 
return SUCCESS
 
static function SQL_CREATEFIELDS( nWA, aStruct )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local n
   local cSchema, cSequenceField, cRddSep
 
   AAdd( aStruct, { FLD_RECNO,   "N", 8, 0 } )   // to emulate DBF recno value
   AAdd( aStruct, { "sql_deleted", "C", 1, 0 } ) // to emulate DBF deleted value
 
   aWAData[ WA_SQLSTRUCT ] := ""
   aWAData[ WA_FCOUNT ]    := Len( aStruct )
 
   s_aStruct := aStruct // hasta que tengamos una conexion real
   cRddSep   := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
 
   for n := 1 to Len( aStruct )
 
      if (aStruct[ n ][ DBS_NAME ] == FLD_RECNO .AND. aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL )
         * Ignore this field into SQL string
      else
         if n > 1
            aWAData[ WA_SQLSTRUCT ] += ", "
         endif
 
         aWAData[ WA_SQLSTRUCT ] += cRddSep + aStruct[ n ][ DBS_NAME ] + cRddSep
      end
 
      do case
         case aStruct[ n ][ DBS_TYPE ] $ "C,Character"
              aWAData[ WA_SQLSTRUCT ] += " CHAR (" + AllTrim( Str( aStruct[ n ][ DBS_LEN ] ) ) + ")"
 
              if aStruct[ n ][ DBS_NAME ] == "sql_deleted"
                 DO CASE
                 CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
                      aWAData[ WA_SQLSTRUCT ] += " DEFAULT 'F'::bpchar NOT NULL"
                 OTHERWISE    
                      aWAData[ WA_SQLSTRUCT ] += " NOT NULL"
                 End
              else   
                 aWAData[ WA_SQLSTRUCT ] += " NULL"
              endif
 
              s_aStruct[ n ][ DBS_TYPE ] = "CHAR" 
 
         case aStruct[ n ][ DBS_TYPE ] == "N"
              do case
                 case aStruct[ n ][ DBS_NAME ] == FLD_RECNO
 
                      DO CASE
                      CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL
                         aWAData[ WA_SQLSTRUCT ] += " BIGINT (15) NOT NULL UNIQUE AUTO_INCREMENT"
                         s_aStruct[ n ][ DBS_TYPE ] = "BIGINT" 
 
                      CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
                         /*
                          * ignore this field ... will be processed in SQL_CREATE() 
                          * because i don't build a corret value for cSequenceField
                          *                        
                         aWAData[ WA_SQLSTRUCT ] += " NUMERIC(15,0) DEFAULT nextval('" +;
                                                          cSequenceField + "'::regclass) UNIQUE NOT NULL"
                         /**/
                         s_aStruct[ n ][ DBS_TYPE ] = "BIGINT" 
                      End
 
                 otherwise     
                      aWAData[ WA_SQLSTRUCT ] += " NUMERIC (" + AllTrim( Str( aStruct[ n ][ DBS_LEN ] ) ) + ")"
                      s_aStruct[ n ][ DBS_TYPE ] = "NUMERIC" 
              endcase
 
         case aStruct[ n ][ DBS_TYPE ] == "L"
              DO CASE
              CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL
                 aWAData[ WA_SQLSTRUCT ] += " TINYINT(1) NOT NULL DEFAULT 0"
 
              CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
                 aWAData[ WA_SQLSTRUCT ] += " BOOLEAN"
 
              OTHERWISE
                 aWAData[ WA_SQLSTRUCT ] += " LOGICAL"               
              End
              s_aStruct[ n ][ DBS_TYPE ] = "LOGICAL" 
 
         case aStruct[ n ][ DBS_TYPE ] == "D"
              aWAData[ WA_SQLSTRUCT ] += " DATE" 
              s_aStruct[ n ][ DBS_TYPE ] = "DATE" 
 
         case aStruct[ n ][ DBS_TYPE ] == "M"
              DO CASE
              CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL
                 aWAData[ WA_SQLSTRUCT ] += " MEDIUMBLOB"
 
              CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
                 aWAData[ WA_SQLSTRUCT ] += " TEXT"
 
              OTHERWISE
                 aWAData[ WA_SQLSTRUCT ] += " unknown_column_type "               
              End
              s_aStruct[ n ][ DBS_TYPE ] = "MEMO" 
 
      endcase
   next
 
return SUCCESS
 
static function SQL_OPEN( nWA, aOpenInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cName, aField, oError, nResult
   local oRecordSet, nTotalFields, n
 
   // When there is no ALIAS we will create new one using file name
   if aOpenInfo[ UR_OI_ALIAS ] == nil
      HB_FNAMESPLIT( aOpenInfo[ UR_OI_NAME ], , @cName )
      aOpenInfo[ UR_OI_ALIAS ] := cName
   endif
 
   cName := SQLAdjustFn( cName ) 
 
   // aWAData[ WA_CONNECTION ] := TOleAuto():New( "ADODB.Connection" )
   aWAData[ WA_TABLENAME ] := cName // s_cTableName
   aWAData[ WA_QUERY ]     := s_cQuery
   aWAData[ WA_USERNAME ]  := s_cUserfName
   aWAData[ WA_PASSWORD ]  := s_cPassword
   aWAData[ WA_SERVER ]    := s_cServer
   aWAData[ WA_ENGINE ]    := s_cEngine
 
   do case
      case Lower( Right( aOpenInfo[ UR_OI_NAME ], 4 ) ) == ".mdb"
           if Empty( aWAData[ WA_PASSWORD ] )
              aWAData[ WA_CONNECTION ]:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + aOpenInfo[ UR_OI_NAME ] )
           else
              aWAData[ WA_CONNECTION ]:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + aOpenInfo[ UR_OI_NAME ] + ";Jet OLEDB:Database Password=" + AllTrim( aWAData[ WA_PASSWORD ] ) )
           endif
 
      case Lower( Right( aOpenInfo[ UR_OI_NAME ], 4 ) ) == ".xls"
           aWAData[ WA_CONNECTION ]:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + aOpenInfo[ UR_OI_NAME ] + ";Extended Properties='Excel 8.0;HDR=YES';Persist Security Info=False" )
 
      case Lower( Right( aOpenInfo[ UR_OI_NAME ], 4 ) ) == ".dbf"
           aWAData[ WA_CONNECTION ]:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + aOpenInfo[ UR_OI_NAME ] + ";Extended Properties=dBASE IV;User ID=Admin;Password=;" )
 
      case Lower( Right( aOpenInfo[ UR_OI_NAME ], 3 ) ) == ".db"
           aWAData[ WA_CONNECTION ]:Open( "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=" + aOpenInfo[ UR_OI_NAME ] + ";Extended Properties='Paradox 3.x';" )
 
      case aWAData[ WA_ENGINE ] == "MYSQL"      // Adjust to use native DLL
           aWAData[ WA_CONNECTION ]:Open( "DRIVER={MySQL ODBC 3.51 Driver};" + ;
                                          "server=" + aWAData[ WA_SERVER ] + ;
                                          ";database=" + aOpenInfo[ UR_OI_NAME ] + ;
                                          ";uid=" + aWAData[ WA_USERNAME ] + ;
                                          ";pwd=" + aWAData[ WA_PASSWORD ] )
           aWAData[ WA_SYSTEMID ] := ID_MYSQL                                           
 
 
      case aWAData[ WA_ENGINE ] == "POSTGRESQL"      // Adjust to use native DLL
//           aWAData[ WA_CONNECTION ]:Open( "DRIVER={MySQL ODBC 3.51 Driver};" + ;
//                                          "server=" + aWAData[ WA_SERVER ] + ;
//                                          ";database=" + aOpenInfo[ UR_OI_NAME ] + ;
//                                          ";uid=" + aWAData[ WA_USERNAME ] + ;
//                                          ";pwd=" + aWAData[ WA_PASSWORD ] )
           aWAData[ WA_SYSTEMID ] := ID_POSTGRESQL                                           
 
      case aWAData[ WA_ENGINE ] == "SQL"
           aWAData[ WA_CONNECTION ]:Open( "Provider=SQLOLEDB;" + ;
                                          "server=" + aWAData[ WA_SERVER ] + ;
                                          ";database=" + aOpenInfo[ UR_OI_NAME ] + ;
                                          ";uid=" + aWAData[ WA_USERNAME ] + ;
                                          ";pwd=" + aWAData[ WA_PASSWORD ] )
 
      case aWAData[ WA_ENGINE ] == "ORACLE"
           aWAData[ WA_CONNECTION ]:Open( "Provider=MSDAORA.1;" + ;
                                          "Persist Security Info=False" + ;
                                          iif( Empty( aWAData[ WA_SERVER ] ),;
                                          "", ";Data source=" + aWAData[ WA_SERVER ] ) + ;
                                          ";User ID=" + aWAData[ WA_USERNAME ] + ;
                                          ";Password=" + aWAData[ WA_PASSWORD ] )
 
   endcase
 
   aWAData[ WA_BOF ] := aWAData[ WA_EOF ] := .F.
 
   aWAData[ WA_FCOUNT ] = If( s_aStruct != nil, Len( s_aStruct ), 0 ) // Hasta que hagamos una conexion real
   UR_SUPER_SETFIELDEXTENT( nWA, nTotalFields := aWAData[ WA_FCOUNT ] )
 
   FOR n := 1 TO nTotalFields
      aField := ARRAY( UR_FI_SIZE )
      aField[ UR_FI_NAME ]    := s_aStruct[ n ][ 1 ]
      aField[ UR_FI_TYPE ]    := SQL_GETFIELDTYPE( s_aStruct[ n ][ 2 ] )
      aField[ UR_FI_TYPEEXT ] := 0
      aField[ UR_FI_LEN ]     := SQL_GETFIELDSIZE( aField[ UR_FI_TYPE ], s_aStruct[ n ][ 2 ] )
      aField[ UR_FI_DEC ]     := 0
 
      /* Insert anothers flags here, such NOT NULL, etc.. (Vailton) */
      UR_SUPER_ADDFIELD( nWA, aField )
    NEXT
 
   nResult := UR_SUPER_OPEN( nWA, aOpenInfo )
 
   if nResult == SUCCESS
      SQL_GOTOP( nWA )
   endif
 
return nResult
 
static function SQL_CLOSE( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /*
   TRY
      oRecordSet:Close()
   CATCH
   END
   */
 
return UR_SUPER_CLOSE( nWA )
 
static function SQL_GETVALUE( nWA, nField, xValue )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local bEmpty  := aWAData[ WA_EOF ]
   local aBuffer, nRow
 
   aBuffer := aWAData[ WA_BUFFER_ARR ]
   nRow    := aWAData[ WA_BUFFER_POS ]
 
   IF !bEmpty .AND. ValType( aBuffer ) == "A"   
      /* We are positioned properly within the buffer? */
      if nRow < 1 .OR. nRow > aWAData[ WA_BUFFER_ROWCOUNT ]
         MsgInfo( "SQL_GETVALUE() -> We are positioned properly within the buffer?" )
         return SUCCESS       
      end
 
      /* This line had been changed before? If not it have NIL value */
      if aBuffer[ nRow ] != NIL
         xValue := aBuffer[ nRow, nField ]
 
         // This is a empty or original value?
         IF (xValue != NIL) 
            return SUCCESS
         End
      end
   End
 
   IF (bEmpty)   
       /* Get formated value */
       DO CASE
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "CHAR"       
            xValue := Space( s_aStruct[ nField ][ DBS_LEN ] )
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "MEMO"
            xValue := ''
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "LOGICAL"       
            xValue := .F.
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "DATE"
            xValue := CTOD('')
 
       OTHERWISE  // Numeric field..
            xValue := 0                        
       End
   ELSE
      /* Extract correct value from original buffer. Native DLL calls here? */
      xValue := nil
   End
 
return SUCCESS
 
static function SQL_GOTOID( nWA, nRecord )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local nRecNo  := nRecord
   local cSQL
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   SQL_RECID( nWA, @nRecNo )
 
   DO CASE
   CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL       
        cSQL = "SELECT " + GetFieldNames( aWAData, .T. ) + " FROM `" + aWAData[ WA_TABLENAME ] + ;
                "` WHERE `"+FLD_RECNO+"` = " + AllTrim( Str( nRecNo ) ) + " // DbGoTo()"
 
   CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL  
        cSQL   := "SELECT " + GetFieldNames( aWAData, .T. ) + " FROM " + SQLGetFullTableName( aWAData ) +;
                   ' WHERE "'+FLD_RECNO+'" = ' + AllTrim( Str( nRecNo ) ) + ' // DbGoTo()'
 
   End
 
   BUFFER_DELETE( aWAData )
   aWAData[ WA_BUFFER_ROWCOUNT ] := 1  // 1 if found, 0 none 
   aWAData[ WA_BUFFER_POS ] := 1       // my row inside aWAData
 
   MsgInfo( cSQL, 'cSQL' )              
 
RETURN If( nRecord == nRecNo, SUCCESS, FAILURE )
 
static function SQL_GOTOP( nWA )
 
   local aWAData    := USRRDD_AREADATA( nWA )
   local cSQL
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   DO CASE
   CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL       
        cSQL := "SELECT " + GetFieldNames( aWAData, .T. ) + " FROM `" + aWAData[ WA_TABLENAME ] + ;
                   "` ORDER BY `"+FLD_RECNO+"` LIMIT 10 // DbGoTop()"
 
   CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL  
        cSQL := "SELECT " + GetFieldNames( aWAData, .T. ) + ' FROM ' + SQLGetFullTableName( aWAData ) +;
                   ' ORDER BY "'+FLD_RECNO+'" LIMIT 10 // DbGoTop()'
 
   End
 
   BUFFER_DELETE( aWAData )
   aWAData[ WA_BUFFER_ROWCOUNT ] := 10 // number of lines retrieved 
   aWAData[ WA_BUFFER_POS ] := 1       // my row inside aWAData   
 
   aWAData[ WA_BOF ] := aWAData[ WA_BUFFER_ROWCOUNT ] < 1
   aWAData[ WA_EOF ] := aWAData[ WA_BUFFER_ROWCOUNT ] < 1
 
   MsgInfo( cSQL )
 
return SUCCESS
 
static function SQL_GOBOTTOM( nWA )
 
   local aWAData    := USRRDD_AREADATA( nWA )
   local cSQL
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   DO CASE
   CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL       
        cSQL := "SELECT " + GetFieldNames( aWAData, .T. ) + " FROM `" + aWAData[ WA_TABLENAME ] + ;
                   "` ORDER BY `"+FLD_RECNO+"` DESC LIMIT 10 // DbGoBottom()"
 
   CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL  
        cSQL := "SELECT " + GetFieldNames( aWAData, .T. ) + ' FROM ' + SQLGetFullTableName( aWAData ) +;
                   ' ORDER BY "'+FLD_RECNO+'" DESC LIMIT 10 // DbGoBottom()'
 
   End
 
   BUFFER_DELETE( aWAData )
   aWAData[ WA_BUFFER_ROWCOUNT ] := 10 // number of lines retrieved 
   aWAData[ WA_BUFFER_POS ] := 1       // my row inside aWAData   
 
   aWAData[ WA_EOF ] := aWAData[ WA_BUFFER_ROWCOUNT ] < 1
   aWAData[ WA_BOF ] := .F.
 
return SUCCESS
 
static function SQL_SKIPRAW( nWA, nRecords )
 
   local aWAData    := USRRDD_AREADATA( nWA )
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
   /*
   if nRecords != 0
      if aWAData[ WA_EOF ]
         if nRecords > 0
            return SUCCESS
         endif
         SQL_GOBOTTOM( nWA )
         ++nRecords
       endif
       if nRecords < 0 // .AND. oRecordSet_AbsolutePosition <= -nRecords
          oRecordSet:MoveFirst()
          aWAData[ WA_BOF ] := .T.
          aWAData[ WA_EOF ] := oRecordSet:EOF
       elseif nRecords != 0
          oRecordSet:Move( nRecords )
          aWAData[ WA_BOF ] := .F.
          aWAData[ WA_EOF ] := oRecordSet:EOF
       endif
   endif
   */
 
return SUCCESS
 
static function SQL_BOF( nWA, lBof )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   lBof := aWAData[ WA_BOF ]
 
return SUCCESS
 
static function SQL_EOF( nWA, lEof )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   lEof := .F. // A determinar cuando hagamos la conexion real
               // meanwhile we don't do a real connection
 
return SUCCESS
 
static function SQL_DELETED( nWA, lDeleted )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
  /*
  TRY
     if oRecordSet:Status == adRecDeleted
        lDeleted := .T.
     else
        lDeleted := .F.
     endif
  CATCH
     lDeleted := .f.
  END
  */
 
return SUCCESS
 
static function SQL_DELETE( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   // Put .T. on sql_deleted here!
   SQL_SKIPRAW( nWA, 1 )
 
return SUCCESS
 
static function SQL_RECNO( nWA, nRecNo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if nRecNo == 0
      nRecNo = aWAData[ WA_RECNO ]
   else
      aWAData[ WA_RECNO ] = nRecNo 
   endif   
 
return SUCCESS
 
static function SQL_RECID( nWA, nRecNo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if nRecNo == 0
      nRecNo = aWAData[ WA_RECNO ]
   else
      aWAData[ WA_RECNO ] = nRecNo 
   endif   
 
return SUCCESS
 
static function SQL_RECCOUNT( nWA, nRecords )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
   local cSQL    := ""
 
   DO CASE
      CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL       
           cSQL := "SELECT COUNT(`"+FLD_RECNO+"`) FROM `" + aWAData[ WA_TABLENAME ] + "`"
 
      CASE aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL  
           cSQL := 'SELECT COUNT("'+FLD_RECNO+'") FROM ' + SQLGetFullTableName( aWAData )
 
   End
 
   IF SET(_SET_DELETED)
      cSQL += " WHERE "+cRddSep+"sql_deleted"+cRddSep+" != 'T' // RecCount()"
   End
 
   nRecords = aWAData[ WA_RECCOUNT ]
 
   MsgInfo( cSQL )
 
return SUCCESS
 
static function SQL_PUTVALUE( nWA, nField, xValue )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
   local lApp    := aWAData[ WA_APPEND ]
   local cSQL    := ''
   local aBuffer, nRow
 
   /*
    * Check if a temporary buffer already exists. (Vailton)
    * 16/09/2008 - 07:49:55
    */
   IF aWAData[ WA_BUFFER_ARR ] == NIL
      IF BUFFER_CREATE( aWAData ) != SUCCESS
         ** Check any error here! (Especially if this function is written in C)
         return FAILURE
      End
   End
 
   aBuffer := aWAData[ WA_BUFFER_ARR ]
   nRow    := aWAData[ WA_BUFFER_POS ] 
 
   /* We are positioned properly within the buffer? */
   if nRow < 1 .OR. nRow > aWAData[ WA_BUFFER_ROWCOUNT ]
      MsgInfo( "SQL_PUTVALUE() -> We are positioned properly within the buffer?" )
      return FAILURE
   end
 
   /* This line had been changed before? If not it have NIL value */
   if aBuffer[ nRow ] == NIL
      aBuffer[ nRow ] := ARRAY( aWAData[ WA_FCOUNT ] )
   end
 
   aBuffer[ nRow, nField ] := xValue
   aWAData[ WA_RECORDCHANGED ] := .T.
return SUCCESS       
 
static function SQL_APPEND( nWA, lUnLockAll )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   HB_SYMBOL_UNUSED( lUnLockAll )
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /* Destroy any buffer data if exists */   
   BUFFER_DELETE( aWAData )
 
   aWAData[ WA_APPEND        ] := .T.
   aWAData[ WA_RECORDCHANGED ] := .T.
   aWAData[ WA_RECCOUNT ]++         // meanwhile we don't do a real connection
 
   /* Create a new buffer for current area*/
   aWAData[ WA_BUFFER_ROWCOUNT]:= 1 // One line... for Append values only
   BUFFER_CREATE( aWAData )
 
   aWAData[ WA_BUFFER_POS ]    := 1
   MsgInfo( "dbAppend()" )
 
return SUCCESS
 
static function SQL_FLUSH( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cSQL := "COMMIT" + " // dbCommitAll()"
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   MsgInfo( cSQL )
 
   if aWAData[ WA_TRANSCOUNTER ] == 0
      // Send commit here..
   end
 
return SUCCESS
 
static function SQL_GOCOLD( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if !aWAData[ WA_RECORDCHANGED ]
      return SUCCESS 
   end   
 
return SQL_WRITERECORD( nWA )
 
static function SQL_ORDINFO( nWA, nIndex, aOrderInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   do case
      case nIndex == UR_ORI_TAG
           // if aOrderInfo[ UR_ORI_TAG ] < aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes:Count
           //    aOrderInfo[ UR_ORI_RESULT ] := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes( aOrderInfo[ UR_ORI_TAG ] ):Name
           // else
              aOrderInfo[ UR_ORI_RESULT ] := ""
           // endif
   endcase
 
return SUCCESS
 
static function SQL_PACK( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
   local cSQL    := "DELETE FROM " + SQLGetFullTableName( aWAData ) + " WHERE "+cRddSep+"sql_deleted"+cRddSep+" = 'T' // __dbPack()"
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   MsgInfo( cSQL )   
 
return SUCCESS
 
static function SQL_RAWLOCK( nWA, nAction, nRecNo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
  HB_SYMBOL_UNUSED( nAction )
  HB_SYMBOL_UNUSED( nRecNo )
 
return SUCCESS
 
static function SQL_LOCK( nWA, aLockInfo  )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   aLockInfo[ UR_LI_METHOD ] := DBLM_MULTIPLE
   aLockInfo[ UR_LI_RECORD ] := RECNO()
   aLockInfo[ UR_LI_RESULT ] := .T.
 
return SUCCESS
 
static function SQL_UNLOCK( nWA, xRecID )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   HB_SYMBOL_UNUSED( xRecID )
 
return SUCCESS
 
static function SQL_SETFILTER( nWA, aFilterInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
 
return SUCCESS
 
static function SQL_CLEARFILTER( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
return SUCCESS
 
static function SQL_ZAP( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cSQL
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   DO CASE
   CASE aWAData[ WA_SYSTEMID ] == ID_MYSQL       
        cSQL := "TRUNCATE TABLE " + SQLGetFullTableName( aWAData ) + " // __dbZap()"
 
   OTHERWISE  
        cSQL := "DELETE FROM " + SQLGetFullTableName( aWAData ) + " // __dbZap()"        
   End
 
   IF aWAData[ WA_SYSTEMID ] == ID_POSTGRESQL
      cSQL += CHR(13) + CHR(10) + 'VACUUM FULL ANALYZE ' + SQLGetFullTableName( aWAData )
   End
 
   MsgInfo( cSQL )      
   aWAData[ WA_RECCOUNT ] = 0 // temporary value meanwhile we don't do a real connection
 
return SUCCESS
 
static function SQL_SETLOCATE( nWA, aScopeInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   aScopeInfo[ UR_SI_CFOR ] := SQLTranslate( aWAData[ WA_LOCATEFOR ] )
 
   aWAData[ WA_SCOPEINFO ] := aScopeInfo
 
return SUCCESS
 
static function SQL_LOCATE( nWA, lContinue )
 
   local aWAData    := USRRDD_AREADATA( nWA )
 
   // USRRDD_SETFOUND( nWA, ! oRecordSet:EOF )
   // aWAData[ WA_EOF ] := oRecordSet:EOF
 
return SUCCESS
 
 
 
static function SQL_CLEARREL( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local nKeys := 0, cKeyName
 
   if aWAData[ WA_CATALOG ] != nil .and. aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys != nil
      TRY
         nKeys := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys:Count
      CATCH
      END
   endif
 
   if nKeys > 0
      cKeyName := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys( nKeys - 1 ):Name
      if !( Upper( cKeyName ) == "PRIMARYKEY" )
         aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys:Delete( cKeyName )
      endif
   endif
 
return SUCCESS
 
static function SQL_RELAREA( nWA, nRelNo, nRelArea )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if nRelNo <= aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys:Count()
      nRelArea := Select( aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys( nRelNo - 1 ):RelatedTable )
   endif
 
return SUCCESS
 
static function SQL_RELTEXT( nWA, nRelNo, cExpr )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if nRelNo <= aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys:Count()
      cExpr := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys( nRelNo - 1 ):Columns( 0 ):RelatedColumn
   endif
 
return SUCCESS
 
static function SQL_SETREL( nWA, aRelInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local cParent := Alias( aRelInfo[ UR_RI_PARENT ] )
   local cChild  := Alias( aRelInfo[ UR_RI_CHILD ] )
   local cKeyName := cParent + "_" + cChild
 
   /*
   TRY
      aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Keys:Append( cKeyName, adKeyForeign,;
                                    aRelInfo[ UR_RI_CEXPR ], cChild, aRelInfo[ UR_RI_CEXPR ] )
   CATCH
      // raise error for can't create relation
   END
   */
 
return SUCCESS
 
static function SQL_ORDLSTADD( nWA, aOrderInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /*
   TRY
      oRecordSet:Index := aOrderInfo[ UR_ORI_BAG ]
   CATCH
   END
   */
 
return SUCCESS
 
static function SQL_ORDLSTCLEAR( nWA )
 
   local aWAData := USRRDD_AREADATA( nWA )
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /*
   TRY
      oRecordSet:Index := ""
   CATCH
   END
   */
 
return SUCCESS
 
static function SQL_ORDCREATE( nWA, aOrderCreateInfo )
 
   local aWAData := USRRDD_AREADATA( nWA )
   local oIndex, oError, n, lFound := .f.
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /*
   if aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes != nil
      for n := 1 to aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes:Count
          oIndex := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes( n - 1 )
          if oIndex:Name == iif( ! Empty( aOrderCreateInfo[ UR_ORCR_TAGNAME ] ), aOrderCreateInfo[ UR_ORCR_TAGNAME ], aOrderCreateInfo[ UR_ORCR_CKEY ] )
             lFound := .T.
             exit
          endif
      next
   endif
 
   TRY
      if aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes == nil .or. ! lFound
         oIndex := TOleAuto():New( "ADOX.Index" )
         oIndex:Name := iif( ! Empty( aOrderCreateInfo[ UR_ORCR_TAGNAME ] ), aOrderCreateInfo[ UR_ORCR_TAGNAME ], aOrderCreateInfo[ UR_ORCR_CKEY ] )
         oIndex:PrimaryKey := .F.
         oIndex:Unique := aOrderCreateInfo[ UR_ORCR_UNIQUE ]
         oIndex:Columns:Append( aOrderCreateInfo[ UR_ORCR_CKEY ] )
         aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes:Append( oIndex )
      endif
   CATCH
      oError := ErrorNew()
      oError:GenCode     := EG_CREATE
      oError:SubCode     := 1004
      oError:Description := HB_LANGERRMSG( EG_CREATE ) + " (" + ;
                            HB_LANGERRMSG( EG_UNSUPPORTED ) + ")"
      oError:FileName    := aOrderCreateInfo[ UR_ORCR_BAGNAME ]
      oError:CanDefault  := .T.
      UR_SUPER_ERROR( nWA, oError )
   END
   */
 
return SUCCESS
 
static function SQL_ORDDESTROY( nWA, aOrderInfo )
 
   local aWAData := USRRDD_AREADATA( nWA ), n, oIndex
 
   if SQL_GOCOLD( nWA ) != SUCCESS
      return FAILURE
   end
 
   /* 
   if aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes != nil
      for n := 1 to aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes:Count
          oIndex := aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes( n - 1 )
          if oIndex:Name == aOrderInfo[ UR_ORI_TAG ]
             aWAData[ WA_CATALOG ]:Tables( aWAData[ WA_TABLENAME ] ):Indexes:Delete( oIndex:Name )
          endif
      next
   endif
   */
 
return SUCCESS
 
function SQLWIN_GETFUNCTABLE( pFuncCount, pFuncTable, pSuperTable, nRddID )
 
   local cSuperRDD   /* NO SUPER RDD */
   local aSQLFunc[ UR_METHODCOUNT ]
 
   aSQLFunc[ UR_INIT ]         := ( @SQL_INIT() )
   aSQLFunc[ UR_NEW ]          := ( @SQL_NEW() )
   aSQLFunc[ UR_CREATE ]       := ( @SQL_CREATE() )
   aSQLFunc[ UR_CREATEFIELDS ] := ( @SQL_CREATEFIELDS() )
   aSQLFunc[ UR_OPEN ]         := ( @SQL_OPEN() )
   aSQLFunc[ UR_CLOSE ]        := ( @SQL_CLOSE() )
   aSQLFunc[ UR_BOF  ]         := ( @SQL_BOF() )
   aSQLFunc[ UR_EOF  ]         := ( @SQL_EOF() )
   aSQLFunc[ UR_DELETED ]      := ( @SQL_DELETED() )
   aSQLFunc[ UR_SKIPRAW ]      := ( @SQL_SKIPRAW() )
   aSQLFunc[ UR_GOTO ]         := ( @SQL_GOTOID() )
   aSQLFunc[ UR_GOTOID ]       := ( @SQL_GOTOID() )
   aSQLFunc[ UR_GOTOP ]        := ( @SQL_GOTOP() )
   aSQLFunc[ UR_GOBOTTOM ]     := ( @SQL_GOBOTTOM() )
   aSQLFunc[ UR_RECID ]        := ( @SQL_RECID() )
   aSQLFunc[ UR_RECNO ]        := ( @SQL_RECNO() )
   aSQLFunc[ UR_RECCOUNT ]     := ( @SQL_RECCOUNT() )
   aSQLFunc[ UR_GETVALUE ]     := ( @SQL_GETVALUE() )
   aSQLFunc[ UR_PUTVALUE ]     := ( @SQL_PUTVALUE() )
   aSQLFunc[ UR_DELETE ]       := ( @SQL_DELETE() )
   aSQLFunc[ UR_APPEND ]       := ( @SQL_APPEND() )
   aSQLFunc[ UR_FLUSH ]        := ( @SQL_FLUSH() )
   aSQLFunc[ UR_GOCOLD ]       := ( @SQL_GOCOLD() )
   aSQLFunc[ UR_ORDINFO ]      := ( @SQL_ORDINFO() )
   aSQLFunc[ UR_PACK ]         := ( @SQL_PACK() )
   aSQLFunc[ UR_RAWLOCK ]      := ( @SQL_RAWLOCK() )
   aSQLFunc[ UR_LOCK ]         := ( @SQL_LOCK() )
   aSQLFunc[ UR_UNLOCK ]       := ( @SQL_UNLOCK() )
   aSQLFunc[ UR_SETFILTER ]    := ( @SQL_SETFILTER() )
   aSQLFunc[ UR_CLEARFILTER ]  := ( @SQL_CLEARFILTER() )
   aSQLFunc[ UR_ZAP ]          := ( @SQL_ZAP() )
   aSQLFunc[ UR_SETLOCATE ]    := ( @SQL_SETLOCATE() )
   aSQLFunc[ UR_LOCATE ]       := ( @SQL_LOCATE() )
   aSQLFunc[ UR_CLEARREL ]     := ( @SQL_CLEARREL() )
   aSQLFunc[ UR_RELAREA ]      := ( @SQL_RELAREA() )
   aSQLFunc[ UR_RELTEXT ]      := ( @SQL_RELTEXT() )
   aSQLFunc[ UR_SETREL ]       := ( @SQL_SETREL() )
   aSQLFunc[ UR_ORDCREATE ]    := ( @SQL_ORDCREATE() )
   aSQLFunc[ UR_ORDDESTROY ]   := ( @SQL_ORDDESTROY() )
   aSQLFunc[ UR_ORDLSTADD ]    := ( @SQL_ORDLSTADD() )
   aSQLFunc[ UR_ORDLSTCLEAR ]  := ( @SQL_ORDLSTCLEAR() )
 
 
return USRRDD_GETFUNCTABLE( pFuncCount, pFuncTable, pSuperTable, nRddID, cSuperRDD,;
                            aSQLFunc )
 
init procedure SQLWIN_INIT()
   rddRegister( "SQLWIN", RDT_FULL )
return
 
static function SQL_GETFIELDSIZE( nDBFFieldType, nSQLFieldSize )
 
   local nDBFFieldSize := 0
 
   do case
 
      case nDBFFieldType == HB_FT_STRING
           nDBFFieldSize := nSQLFieldSize
 
      case nDBFFieldType == HB_FT_INTEGER
           nDBFFieldSize := nSQLFieldSize
 
      case nDBFFieldType == HB_FT_DOUBLE
           nDBFFieldSize := nSQLFieldSize
 
      case nDBFFieldType == HB_FT_DATE
           nDBFFieldSize := 8
 
      case nDBFFieldType == HB_FT_LOGICAL
           nDBFFieldSize := 1
 
      case nDBFFieldType == HB_FT_MEMO
           nDBFFieldSize := 10
 
   endcase
 
return nDBFFieldSize
 
static function SQL_GETFIELDTYPE( nSQLFieldType )
 
   local nDBFFieldType := 0
 
   do case
      case nSQLFieldType == "CHAR"
           nDBFFieldType := HB_FT_STRING
 
      case nSQLFieldType == "LOGICAL"
           nDBFFieldType := HB_FT_LOGICAL
 
      case nSQLFieldType == "NUMERIC"
           nDBFFieldType := HB_FT_INTEGER
 
      case nSQLFieldType == "BIGINT"
           nDBFFieldType := HB_FT_DOUBLE
 
   endcase
 
return nDBFFieldType
 
static function SQLTranslate( cExpr )
 
   if Left( cExpr, 1 ) == '"' .and. Right( cExpr, 1 ) == '"'
      cExpr := SubStr( cExpr, 2, Len( cExpr ) - 2 )
   endif
 
   cExpr := StrTran( cExpr, '""', "" )
   cExpr := StrTran( cExpr, '"', "'" )
   cExpr := StrTran( cExpr, "''", "'" )
   cExpr := StrTran( cExpr, "==", "=" )
   cExpr := StrTran( cExpr, ".and.", "AND" )
   cExpr := StrTran( cExpr, ".or.", "OR" )
   cExpr := StrTran( cExpr, ".AND.", "AND" )
   cExpr := StrTran( cExpr, ".OR.", "OR" )
 
return cExpr
 
function HB_SQLRddGetConnection( nWA )
 
   DEFAULT nWA TO Select()
 
return USRRDD_AREADATA( nWA )[ WA_CONNECTION ]
 
static function GetFieldNames( aWAData, lRecNo )
 
   local cFields := "", n
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
 
   if s_aStruct != nil
      for n = 1 to Len( s_aStruct )
         if lRecNo
            cFields += cRddSep + s_aStruct[ n ][ DBS_NAME ] + cRddSep + ", "
         else
            if s_aStruct[ n ][ DBS_NAME ] != "sql_reno"                            // That field is this? (Vailton)   
               cFields += cRddSep + s_aStruct[ n ][ DBS_NAME ] + cRddSep + ", "
            endif   
         endif
      next      
   endif
 
   cFields = SubStr( cFields, 1, Len( cFields ) - 2 )
 
return cFields      
 
static function GetFieldEmptyValues( aWAData, lRecNo )
 
   local cValues := "", n
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
 
   for n = 1 to Len( s_aStruct )
      if lRecNo
         cValues += cRddSep + Space( s_aStruct[ n ][ DBS_LEN ] ) + cRddSep + ", "
      else
         if s_aStruct[ n ][ DBS_NAME ] != "sql_reno"   
            cValues += cRddSep + Space( s_aStruct[ n ][ DBS_LEN ] ) + cRddSep + ", "    // That field is this? (Vailton)
         endif   
      endif   
   next
 
   cValues = SubStr( cValues, 1, Len( cValues ) - 2 )
 
return cValues      
 
/*
 * Return the correct DBMS separator
 * 15/09/2008 - 21:25:46
 */
static function SQLGetCurrentSep( nSysID )
      DO CASE
      CASE nSysID == ID_MYSQL       ; return "`"
      CASE nSysID == ID_POSTGRESQL  ; return '"'
      End
   return ''
 
/*
 * Return a full-qualified tablename
 * 15/09/2008 - 21:49:27
 */
static function SQLGetFullTableName( aWAData )
   local cSchema    := 'public'
   local cTableName := aWAData[ WA_TABLENAME ]
   local nSysID     := aWAData[ WA_SYSTEMID ]
 
      DO CASE
      CASE nSysID == ID_MYSQL
         return "`" + cTableName + "`"
 
      CASE nSysID == ID_POSTGRESQL
         return '"' + cSchema + '"."' + cTableName + '"'
      End
 
   return cTableName
 
/*
 * Return the correct name for a table
 * 15/09/2008 - 21:25:46
 */
function SQLAdjustFn( cDataBase )
   if "." $ cDataBase
      cDataBase = StrTran( cDataBase, ".", "_" )
   else
      cDataBase += "_dbf"
   endif      
   return cDataBase
 
/*
 * Create a new buffer with defined size. (Vailton)
 * 16/09/2008 - 07:56:05
 */
static function BUFFER_CREATE( aWAData )
 
   if VALTYPE( aWAData[ WA_BUFFER_ARR ] ) == 'A'
      return SUCCESS
   end
 
   aWAData[ WA_BUFFER_ARR ] := Array( aWAData[ WA_BUFFER_ROWCOUNT ] )
   return SUCCESS
 
/*
 * Create destroy a buffer (if exists) into current workarea. (Vailton)
 * 16/09/2008 - 08:04:50
 */
static function BUFFER_DELETE( aWAData )
 
   aWAData[ WA_BUFFER_ARR]  := NIL
 
   return SUCCESS
 
/*
 * Write current record into server with correct SQL command. Based on lAppend
 * to build INSERT / UPDATE statements.
 * 16/09/2008 - 08:13:32
 */    
static function SQL_WRITERECORD( nWA )
   local aWAData := USRRDD_AREADATA( nWA )
   local cRddSep := SQLGetCurrentSep( aWAData[ WA_SYSTEMID ] )
   local nSysID  := aWAData[ WA_SYSTEMID ]                
   local lApp    := aWAData[ WA_APPEND ]
   local cSQL    := ''
   local cTableName, aBuffer, nRow, nField, nRecNo
   local nCount, Temp
 
   IF aWAData[ WA_BUFFER_ARR ] == NIL
      return SUCCESS
   End
 
   aBuffer := aWAData[ WA_BUFFER_ARR ]
   nRow    := aWAData[ WA_BUFFER_POS ] 
 
   /* We are positioned properly within the buffer? */
   if nRow < 1 .OR. nRow > aWAData[ WA_BUFFER_ROWCOUNT ]
      return FAILURE
   end
 
   /* This line had been changed before? If not it have NIL value */
   if aBuffer[ nRow ] == NIL
      return SUCCESS
   end
 
   SQL_RECID( nWA, @nRecNo )   
 
   if ( lApp )
      cSQL += "INSERT INTO " + SQLGetFullTableName( aWAData ) + " SET "
   else
      cSQL += "UPDATE " + SQLGetFullTableName( aWAData ) + " SET "
   end
 
   nCount := 0
 
   for nField = 1 to Len( s_aStruct )
 
       if (aBuffer[ nRow, nField ] == NIL)
          loop
       end
       nCount ++
 
       /* Get the current fieldname */  
       cSQL += iif( nCount>1, ", ", "" ) + cRddSep + s_aStruct[ nField ][ DBS_NAME ] + cRddSep + " = "       
 
       /* Get formated value */
       DO CASE
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "CHAR"
 
            /* Test correct field length to avoid some errors */            
            if Len(aBuffer[ nRow, nField ]) <= s_aStruct[ nField ][ DBS_LEN ]
               cSQL += "'" + StrTran( aBuffer[ nRow, nField ], "'", "\'" ) + "'"    // StrTran() to emulate escape
            else
               cSQL += "'" + StrTran( LEFT( aBuffer[ nRow, nField ], s_aStruct[ nField ][ DBS_LEN ]), "'", "\'" ) + "'"   
            end
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "MEMO"
 
            cSQL += "'" + StrTran( aBuffer[ nRow, nField ], "'", "\'" ) + "'"    // StrTran() to emulate escape
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "LOGICAL"
 
            DO CASE
            CASE nSysID == ID_MYSQL
                 cSQL += IIF( aBuffer[ nRow, nField ], "1", "0" )                // http://dev.mysql.com/doc/refman/5.1/en/numeric-type-overview.html
 
            CASE nSysID == ID_POSTGRESQL
                 cSQL += IIF( aBuffer[ nRow, nField ], "TRUE", "FALSE" )         // http://www.postgresql.org/docs/8.1/interactive/datatype-boolean.html
 
            OTHERWISE
                 cSQL += "'T'"   // hummm.... ???
            End
 
       CASE s_aStruct[ nField ][ DBS_TYPE ] == "DATE"
 
            IF Empty( aBuffer[ nRow, nField ] )
               cSQL += "NULL"
            ELSE  
               cSQL += DTOS( aBuffer[ nRow, nField ] )
            End
 
       OTHERWISE  // Numeric field..
            cSQL += AllTrim( Str( aBuffer[ nRow, nField ] ) )            
 
       End
   next
 
   IF ! lApp .and. RecNo() != nil
      cSQL += " WHERE " + cRddSep + FLD_RECNO + cRddSep + " = " + AllTrim( Str( RecNo() ) )
   End
 
   aWAData[ WA_APPEND        ] := .F.
   aWAData[ WA_RECORDCHANGED ] := .F.
 
   MsgInfo( cSQL )
 
   if lApp
      /*
       * Update sql_recno where:
       *
      mysql_insert_id(pMysql);
      sprintf( str, "select %s", pPostgreSQL->szSequenceName ); 
      /**/
 
      /*
       * WARNING: If a current indexed field (indexkey()) is changed, this buffer
       * must be invalid!!
       *
      IF ...
         aWAData[ WA_INVALIDBUFFER ] := .T.     // To force load correct values
      End
      /**/ 
   end      
return SUCCESS
```