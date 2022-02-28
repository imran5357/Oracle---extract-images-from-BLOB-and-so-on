# Oracle---extract-images-from-BLOB-and-so-on

#Create directory:
<pre>
Create or replace directory MY_IMAGES as '/tmp/my_images';
Grant read, write on directory MY_IMAGES to BOB;
</pre>

#create Package specification:

<pre>
Create or replace package fileutils as

   --
   -- This procedure deletes a file, and depends on an Oracle DIRECTORY object being passed
   --
   Procedure delete_os_file (i_directory varchar2, i_filename varchar2);

   --
   -- This procedure moves and optionally renames a file, 
   -- and depends on an Oracle DIRECTORY object being passed
   --
   Procedure move_os_file ( i_source_directory in varchar2, i_source_file in varchar2, i_target_directory in varchar2, i_target_file in varchar2);

   --
   -- This procedure takes a blob variable and writes it to a file, 
   -- and depends on an Oracle DIRECTORY object being passed
   --
   Procedure blob_into_file (i_directory in varchar2, i_file_name in varchar2, i_blob in blob);

   --
   -- This procedure takes a file and uploads it into a blob variable
   -- and depends on an Oracle DIRECTORY object being passed
   --
   Procedure file_into_blob(i_directory in varchar2, i_file_name in varchar2, o_blob out blob);

   --
   -- This procedure converts a clob to a blob
   --
   Procedure convert_clob_to_blob (i_clob in clob, o_blob out blob);

   --
   -- This procedure converts a blob to a clob
   --
   Procedure convert_blob_to_clob (i_blob in blob, o_clob out clob);

   --
   -- This one checks for file existence without Java
   --
   Function file_exists (i_directory in varchar2, i_filename in varchar2) return boolean;

   --
   -- Returns the basename of a filename
   -- Works with Windows and UNIX pathnames
   --
   Function basename (i_filename in varchar2) return varchar2;

   --
   -- This takes a Base64 string and converts it to a binary BLOB
   --
   Procedure base64_string_to_blob (i_clob in clob, o_blob out blob);
   Function base64_string_to_blob (i_clob in clob) return blob;

   --
   -- This takes a binary BLOB and converts it to a Base64 string
   --
   Procedure blob_to_base64_string (i_blob in blob, o_clob out clob);
   Function blob_to_base64_string (i_blob in blob) return clob;

End fileutils;
/
<pre>


#Package body

<pre>
Create or replace package body fileutils as

   Procedure delete_os_file (i_directory varchar2, i_filename varchar2)
   is

   Begin

      utl_file.fremove(i_directory,i_filename);

   End;

   Procedure move_os_file
   (
      i_source_directory     in varchar2,
      i_source_file          in varchar2,
      i_target_directory     in varchar2,
      i_target_file          in varchar2
   )

   is

      srcdir               varchar2(255) := upper(i_source_directory);
      tgtdir               varchar2(255) := upper(i_target_directory);

   begin

      --
      -- NOTE: If you're getting the all-too-familiar
      -- ORA-29292: file rename operation failed
      -- and you're SURE that your directory names are correct,
      -- and you're SURE that your privileges are correct, both at the
      -- OS level, and within the database, there's one last thing that
      -- can get you. I learned the hard way that this command will NOT
      -- work successfully renaming a file from one filesystem to another,
      -- at least when those filesystems are NFS mounted. That is all.
      --

      utl_file.frename(srcdir,i_source_file,tgtdir,i_target_file,TRUE);

   end move_os_file;

   Procedure blob_into_file (i_directory in varchar2, i_file_name in varchar2, i_blob in blob)
   is

      l_file            utl_file.file_type;
      l_buffer          raw(32767);
      l_amount          binary_integer := 32767;
      l_pos             integer := 1;
      i_blob_len        integer;

   Begin

      i_blob_len := dbms_lob.getlength(i_blob);
      l_pos:= 1;

      -- Open the destination file.
      l_file := utl_file.fopen(i_directory,i_file_name,'wb', 32767);

      -- Read chunks of the BLOB and write them to the file
      -- until complete.
      while l_pos < i_blob_len loop
         dbms_lob.read(i_blob, l_amount, l_pos, l_buffer);
         utl_file.put_raw(l_file, l_buffer, TRUE);
         l_pos := l_pos + l_amount;
      end loop;

      -- Close the file.
      utl_file.fclose(l_file);

   End blob_into_file;

   Procedure file_into_blob(i_directory in varchar2, i_file_name in varchar2, o_blob out blob) 
   is
      src_loc       bfile   := bfilename(i_directory, i_file_name);
   Begin

      -- Initialize the dest blob
      o_blob := empty_blob();

      -- Open source binary file from OS
      dbms_lob.open(src_loc, dbms_lob.lob_readonly);

      -- Create temporary LOB object
      dbms_lob.createtemporary(
            lob_loc => o_blob
          , cache   => true
          , dur     => dbms_lob.session
      );

      -- Open temporary lob
      dbms_lob.open(o_blob, dbms_lob.lob_readwrite);

      -- Load binary file into temporary LOB
      dbms_lob.loadfromfile(
            dest_lob => o_blob
          , src_lob  => src_loc
          , amount   => dbms_lob.getLength(src_loc));

      -- Close lob objects
      dbms_lob.close(o_blob);
      dbms_lob.close(src_loc);

   End file_into_blob;

   Function basename (i_filename in varchar2) return varchar2
   is
      v_basename        varchar2(1024);
   Begin

      --
      -- If the regex's below don't match, then it's already at its base name
      -- Return what was passed.
      --
      v_basename := i_filename;

      if regexp_like(i_filename,'^.*\\') then
         dbms_output.put_line('This is a Windows file');
         v_basename := regexp_substr(i_filename,'[^\]*$');
         dbms_output.put_line('Basename is : '||v_basename);
      end if;
      if regexp_like(i_filename,'^/') then
         dbms_output.put_line('This is a UNIX file');
         v_basename := regexp_substr(i_filename,'[^/]*$');
         dbms_output.put_line('Basename is : '||v_basename);
      end if;

      return v_basename;

   End basename;

   Function file_exists (i_directory in varchar2, i_filename in varchar2) return boolean
   is
      v_exists          boolean;
      v_file_length     number;
      v_block_size      number;
   Begin
      utl_file.fgetattr(upper(i_directory), i_filename, v_exists, v_file_length, v_block_size);   
      if (v_exists) then
         dbms_output.put_line('File '||i_filename||' exists, '||v_file_length||' bytes');
      else
         dbms_output.put_line('File '||i_filename||' does not exist');
      end if;

      return v_exists;

   end file_exists;

   Procedure convert_clob_to_blob (i_clob in clob, o_blob out blob)
   is

      v_in      pls_Integer := 1;
      v_out     pls_Integer := 1;
      v_lang    pls_Integer := 0;
      v_warning pls_Integer := 0;

   Begin

      dbms_lob.createtemporary(o_blob,TRUE);
      dbms_lob.converttoblob(o_blob,i_clob,DBMS_lob.getlength(i_clob),v_in,v_out,dbms_lob.default_csid,v_lang,v_warning);

   End convert_clob_to_blob;

   Procedure convert_blob_to_clob (i_blob in blob, o_clob out clob)
   is

      v_in      pls_Integer := 1;
      v_out     pls_Integer := 1;
      v_lang    pls_Integer := 0;
      v_warning pls_Integer := 0;

   Begin

      dbms_lob.createtemporary(o_clob,TRUE);
      dbms_lob.converttoclob(o_clob,i_blob,DBMS_lob.getlength(i_blob),v_in,v_out,dbms_lob.default_csid,v_lang,v_warning);

   End convert_blob_to_clob;

   Procedure blob_to_base64_string (i_blob in blob, o_clob out clob)
   is

      v_out_cl     clob;
      file_len     pls_integer;
      modulo       pls_integer;
      pieces       pls_integer;
      amt          binary_integer      := 23808;
      buf          raw (32767);
      buf_tx       varchar2(32767);
      pos          pls_integer         := 1;
      filepos      pls_integer         := 1;
      counter      pls_integer         := 1;
   Begin
      dbms_lob.createtemporary (v_out_cl, true, dbms_lob.call);
      file_len := dbms_lob.getlength (i_blob);
      modulo := mod (file_len, amt);
      pieces := trunc (file_len / amt);

      while (counter <= pieces) loop
         dbms_lob.read (i_blob, amt, filepos, buf);
         buf_tx:=utl_raw.cast_to_varchar2 (utl_encode.base64_encode (buf));
         dbms_lob.writeappend (v_out_cl,length(buf_tx),buf_tx);
         filepos := counter * amt + 1;
         counter := counter + 1;
      end loop;

      if (modulo <> 0) THEN
         dbms_lob.read (i_blob, modulo, filepos, buf);
         buf_tx:=utl_raw.cast_to_varchar2 (utl_encode.base64_encode (buf));
         dbms_lob.writeappend (v_out_cl,length(buf_tx),buf_tx);
      end if;

      o_clob := v_out_cl;

   End blob_to_base64_string;

   Function blob_to_base64_string (i_blob in blob) return clob
   is
      v_out_cl     clob;
      file_len     pls_integer;
      modulo       pls_integer;
      pieces       pls_integer;
      amt          binary_integer      := 23808;
      buf          raw (32767);
      buf_tx       varchar2(32767);
      pos          pls_integer         := 1;
      filepos      pls_integer         := 1;
      counter      pls_integer         := 1;
   Begin

      dbms_lob.createtemporary (v_out_cl, true, dbms_lob.call);
      file_len := dbms_lob.getlength (i_blob);
      modulo := mod (file_len, amt);
      pieces := trunc (file_len / amt);

      while (counter <= pieces) loop
         dbms_lob.read (i_blob, amt, filepos, buf);
         buf_tx:=utl_raw.cast_to_varchar2 (utl_encode.base64_encode (buf));
         dbms_lob.writeappend (v_out_cl,length(buf_tx),buf_tx);
         filepos := counter * amt + 1;
         counter := counter + 1;
      end loop;

      if (modulo <> 0) THEN
         dbms_lob.read (i_blob, modulo, filepos, buf);
         buf_tx:=utl_raw.cast_to_varchar2 (utl_encode.base64_encode (buf));
         dbms_lob.writeappend (v_out_cl,length(buf_tx),buf_tx);
      end if;

      return v_out_cl;

   End blob_to_base64_string;

   Procedure base64_string_to_blob (i_clob in clob, o_blob out blob)
   is

      v_out_bl blob;
      clob_size number;
      pos number;
      charBuff varchar2(32767);
      dBuffer RAW(32767);
      v_readSize_nr number;
      v_line_nr number;

   begin
      dbms_lob.createTemporary (v_out_bl, true, dbms_lob.call);
      v_line_nr:=greatest(65, instr(i_clob,chr(10)), instr(i_clob,chr(13)));
      v_readSize_nr:= floor(32767/v_line_nr)*v_line_nr;
      clob_size := dbms_lob.getLength(i_clob);
      pos := 1;

      while (pos < clob_size) loop
         dbms_lob.read (i_clob, v_readSize_nr, pos, charBuff);
         dBuffer := UTL_ENCODE.base64_decode (utl_raw.cast_to_raw(charBuff));
         dbms_lob.writeAppend (v_out_bl,utl_raw.length(dBuffer),dBuffer);
         pos := pos + v_readSize_nr;
      end loop;

      o_blob := v_out_bl;

   end base64_string_to_blob;

   Function  base64_string_to_blob (i_clob in clob) return blob
   is

      v_out_bl blob;
      clob_size number;
      pos number;
      charBuff varchar2(32767);
      dBuffer RAW(32767);
      v_readSize_nr number;
      v_line_nr number;

   begin
      dbms_lob.createTemporary (v_out_bl, true, dbms_lob.call);
      v_line_nr:=greatest(65, instr(i_clob,chr(10)), instr(i_clob,chr(13)));
      v_readSize_nr:= floor(32767/v_line_nr)*v_line_nr;
      clob_size := dbms_lob.getLength(i_clob);
      pos := 1;

      while (pos < clob_size) loop
         dbms_lob.read (i_clob, v_readSize_nr, pos, charBuff);
         dBuffer := UTL_ENCODE.base64_decode (utl_raw.cast_to_raw(charBuff));
         dbms_lob.writeAppend (v_out_bl,utl_raw.length(dBuffer),dBuffer);
         pos := pos + v_readSize_nr;
      end loop;

      return v_out_bl;

   end base64_string_to_blob;

end fileutils;
/
</pre>
