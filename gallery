#!/usr/bin/env python
"""
album.py -- Generate an HTML gallery from a directory of jpg files.
Requires Python 2.4 or greater.
"""

#*****************************************************************************
#       Copyright (C) 2005 William Stein <was@math.harvard.edu>
#  (except for the exif reading code included near the middle of this file!)
#
#  Distributed under the terms of the GNU General Public License (GPL)
#
#    This code is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
#    General Public License for more details.
#
#  The full text of the GPL is available at: http://www.gnu.org/licenses/
#*****************************************************************************

import cPickle, os

PHOTO_EXTENSIONS=["jpg","jpeg", 'png']
VIDEO_EXTENSIONS=["avi","mpg"]
EXTENSIONS = PHOTO_EXTENSIONS + VIDEO_EXTENSIONS
VIEWER = "gqview"
PROPS = ['author', 'title', 'date', 'email', 'bgcolor',
         'columns', 'thumb_size', 'thumb_quality',
         'medium_size', 'medium_quality', 'textcolor',
         'bordercolor']
DEFAULT_QUALITY=99
PROPS.sort()

class lazy_prop(object):
    def __init__(self, calculate_function):
        self._calculate = calculate_function
        self.__doc__ = calculate_function.__doc__

    def __get__(self, obj, _=None):
        if obj is None:
            return self
        value = self._calculate(obj)
        setattr(obj, self._calculate.func_name, value)
        return value

def prop(f):
    return property(f, None, None, f.__doc__)

class Photo(object):
    def __init__(self, filename, caption="", rating=0):
        self.__filename = filename
        self.caption = caption
        self.rating = rating

    def __repr__(self):
        s = "%s: %s"%(self.name, self.rating)
        if len(self.caption) > 0:
            s += " %s"%self.caption
        return s 

    def __set_caption(self, x):
        x = str(x)
        if x.find(":") != -1:
            raise ValueError, "Captions cannot contain a colon (i.e., ':')."
        self.__caption = x
        
    def __get_caption(self):
        return self.__caption
    caption = property(__get_caption, __set_caption)

    def __set_rating(self, x):
        self.__rating = int(x)
        
    def __get_rating(self):
        return self.__rating
    rating = property(__get_rating, __set_rating)

    @lazy_prop
    def is_video(self):
        return self.ext.lower() in VIDEO_EXTENSIONS

    @lazy_prop
    def is_photo(self):
        return self.ext.lower() in PHOTO_EXTENSIONS

    @prop
    def filename(self):
        return self.__filename

    @lazy_prop
    def ext(self):
        i = self.name.rfind(".")
        if i == -1:
            return ""
        return self.name[i+1:]

    @lazy_prop
    def base(self):
        i = self.name.rfind(".")
        if i == -1:
            return self.name
        return self.name[:i]

    @lazy_prop
    def path(self):
        return os.path.split(self.filename)[0]

    @lazy_prop
    def name(self):
        return os.path.split(self.filename)[1]

    def __cmp__(self, other):
        """
        Compare based on filename.

        Date would be nice, but camera clocks are
        often wrong...
        
        """
        if not isinstance(other, Photo):
            return -1
        return cmp(self.filename, other.filename)
        
##         if self.is_photo and other.is_video:
##             return -1
##         elif self.is_video and other.is_photo:
##             return 1
##         if self.filename == other.filename:
##             return 0
##         if self.datetime < other.datetime:
##             return -1
##         if self.datetime > other.datetime:
##             return 1
##         if self.filename < other.filename:
##             return -1
##         elif self.filename > other.filename:
##             return 1
##         assert False
        
    ########################################################
    ## Extraction of exif information from photo file
    ########################################################

    @lazy_prop
    def exif(self):
        cached = "%s/.exif/%s.exif"%(self.path, self.base)
        if os.path.exists(cached):
            return cPickle.load(open(cached))
        print "Extracting exif information from %s"%self.name
        x = process_file(open(self.filename,"rb"))
        if not os.path.exists("%s/.exif"%self.path):
            os.mkdir("%s/.exif"%self.path)
        y = {}
        for k in x.keys():
            if len(str(x[k])) < 500:
                y[k] = x[k]
        cPickle.dump(y, open(cached,"w"))
        return x

    @prop
    def camera(self):
        try:
            return str(self.exif["Image Model"])
        except:
            return "Unknown model"

    @prop
    def comment(self):
        try:
            return str(self.exif["EXIF UserComment"])
        except:
            return ""

    @prop
    def flash(self):
        try:
            return self.exif["MakerNote FlashMode"]
        except:
            return ""

    @lazy_prop
    def focal_length(self):
        """
        The focal length in millimeters.
        """
        try:
            return str(self.exif["EXIF FocalLength"])
        except:
            return "?"

    @lazy_prop
    def iso(self):
        try:
            return int(str(self.exif["EXIF ISOSpeedRatings"]))
        except:
            return '?'

    @prop
    def metering_mode(self):
        try:
            return self.exif["MakerNote MeteringMode"]
        except:
            return "Unknown"

    @lazy_prop
    def orientation(self):
        """
        The orientation of the photo, which is an int:
            1 -- normal landscape
            6 -- photo needs to be rotated 90 degrees clockwise in 
                 order to look normal.
            8 -- photo needs to be rotated 90 degrees counterclockwise
                 to look normal.
        """
        try:
            return int(str(self.exif["Image Orientation"]))
        except:
            return 0

    @lazy_prop
    def needed_rotation(self):
        if self.orientation == 6:
            return 90
        elif self.orientation == 8:
            return 270
        else:
            return 0

    @prop
    def quality(self):
        try:
            return self.exif["MakerNote Quality"]
        except:
            return ""

    @lazy_prop
    def width(self):
        try:
            return eval(str(self.exif["EXIF ExifImageWidth"]))
        except:
            return -1

    @lazy_prop
    def height(self):
        try:
            return eval(str(self.exif["EXIF ExifImageLength"]))
        except:
            return -1

    @lazy_prop
    def pixels(self):
        try:
            return self.width * self.height
        except:
            return -1

    @lazy_prop
    def megapixels(self):
        try:
            return self.pixels / (10.0**6)
        except:
            return -1

    @lazy_prop
    def shutter_speed(self):
        try:
            return str(self.exif["EXIF ExposureTime"])
        except:
            return "-1"

    @lazy_prop
    def f_number(self):
        try:
            s = str(self.exif["EXIF FNumber"])
            i = s.find("/")
            assert s[i+1:] == "10"
            s = s[:i]
            if s[-1] == "0":
                return s[:-1]
            else:
                return s[:-1] + "." + s[-1]
        except:
            return "?"

    @prop
    def aperature(self):
        try:
            return "1/%s"%self.f_number
        except:
            return "?"

    @lazy_prop
    def datetime(self):
        if self.is_video:
            return ""
        else:
            try:
                s = str(self.exif["EXIF DateTimeOriginal"]).split()
                return s[0].replace(":","/") + ", " + s[1]
            except:
                return "?"

    @prop
    def white_balance(self):
        try:
            return str(self.exif["MakerNote WhiteBalance"])
        except:
            return "?"

    @prop
    def sharpness(self):
        try:
            return str(self.exif["MakerNote Sharpness"])
        except:
            return "?"

    #@prop
    #def af_point(self):
    #    return self.exif["MakerNote AFPointSelected"]
    
    @prop
    def long_focal_length_in_focal_units(self):
        try:
            return int(self.exif["MakerNote LongFocalLengthOfLensInFocalUnits"])
        except:
            return "?"

    @prop
    def short_focal_length_in_focal_units(self):
        try:
            return int(self.exif["MakerNote ShortFocalLengthOfLensInFocalUnits"])
        except:
            return "?"
    

    ########################################################
    ## Generate resized image of given size in given directory.
    ########################################################
    def small(self, size, rotate=True, quality=DEFAULT_QUALITY):
        dir = "%s/.small"%self.path
        if os.path.exists(dir) and not os.path.isdir(dir):
            os.remove(dir)
        if not os.path.exists(dir):
            os.mkdir(dir)
        file = "%s/%s-small-%s-%s.jpg"%(dir, self.base, size, quality)
        if not os.path.exists(file):
            if rotate:
                rot = self.needed_rotation
            else:
                rot = 0
            cmd = 'convert -rotate %s -quality %s -size %sx%s "%s" \
-resize %sx%s +profile "*" "%s"'%\
            (rot, quality, size, size, self.filename, size, size, file)
            print "Resizing %s to %s."%(self.name, size)
            os.system(cmd)
        return file

    ########################################################
    ## Generate HTML representation for this image in
    ## the directory dir, under os.curdir
    ########################################################
    def html(self, album, dir, prev, next, medium_size, quality=DEFAULT_QUALITY, original=True):
        if self.is_photo:
            med = self.small(size=medium_size, rotate=True, quality=quality)
            medium = "%s-medium.jpg"%self.base
            os.symlink("../" + med, dir + "/" + medium)
        td = '<TD bgcolor="%s">'%album.bgcolor
        tdwhite = '<TD bgcolor="%s">'%'white'
        body = ""
        body += '<H1><A href="index.html">%s</A></H1>\n'%album.title
        #body += '<H2>%s</H2>\n'%self.name
        body += '<TABLE width=%s border=0 cellpadding=6 cellspacing=2 bgcolor="%s">\n'%(medium_size, album.bordercolor)
        body += '<TR>%s<A href="%s.html">PREV</A></TD>\n'%(tdwhite,prev.base)
        body += '%s<A href="%s.html">NEXT</A></TD>\n'%(tdwhite,next.base)
        body += '%s<A href="index.html">UP</A></TD>\n'%tdwhite
        if self.is_photo:
            body += '%s<FONT size=-1 color="%s">%s, %smm, %ss, f/%s, ISO%s, %s</FONT></TD>\n'%(
                td, album.textcolor, self.datetime,
                self.focal_length, self.shutter_speed,
                self.f_number, self.iso, self.camera)
        body += '%s<FONT size=0 color="%s">Rating: %s</TD>\n'%(td, album.textcolor, self.rating)
        if original or self.is_video:
            body += '%s<A HREF="%s">%s</A></TD>\n'%(tdwhite, self.name, self.name)
            os.symlink("../" + self.filename, dir + "/" + self.name)
        body += '</TR></TABLE>\n\n'
        if len(self.caption) > 0:
            body += '<TABLE border=0 cellpadding=6 width=%s bgcolor="%s"><TR><TD bgcolor="%s"><FONT color=%s>%s</FONT></TD></TR></TABLE>'%(
                medium_size, album.bordercolor, album.bgcolor, album.textcolor, self.caption)
        body += '<TABLE border=0 cellpadding=4 cellspacing=2 bgcolor="%s"><TR>\n'%album.bgcolor
        #body += '%s<A href="%s.html"><IMG src="%s-thumb.jpg"></A></td>\n'%(td, prev.base, prev.base)
        if self.is_photo:
            body += '<TD bgcolor="%s"><A href="%s.html"><IMG src="%s"></A></td>\n'%(album.bgcolor,next.base, medium)
        else:
            body += '%s<A HREF="%s.html"><TABLE width=%s bgcolor=%s cellpadding=8><tr><td align=center bgcolor=white>VIDEO CLIP (<A HREF="%s.html">NEXT</A>)</td></tr>  \
                    <tr><td align=center  bgcolor=white><A href="%s" alt="%s">%s</A></td></tr></table></A>\
                    \n'%(td, next.base, album.medium_size, album.bordercolor, next.base, self.name, self.caption, self.name)
        #body += '%s<A href="%s.html"><IMG src="%s-thumb.jpg"></A></td>\n'%(td,next.base, next.base)
        body += "</TR></TABLE>\n"
        if hasattr(album, 'author'):
            body += '<hr>%s'%album.author_link
        

        s = """
        <HTML>
        <HEAD>
           <TITLE>%s</TITLE>
        </HEAD>
        <BODY bgcolor="%s">
           <FONT color="%s">
           <DIV align=center>
           %s
           </DIV>
           </FONT>
        </BODY>
        """%(album.title, album.bgcolor, album.textcolor, body)
        
        open('%s/%s.html'%(dir, self.base), "w").write(s)

        

    ########################################################
    ## View the photo
    ########################################################    
    def show(self):
        os.system("%s %s&"%(VIEWER, self.filename))


class Album(object):
    def __init__(self, photos=[], title="", derived_from=None):
        """
        photos -- list of objects of type Photo
        title -- title of this album
        """
        self.__photos = list(photos)
        self.__photos.sort()   
        # photo_dict provides alternative dictionary access to the photos
        self.__create_photo_dict()
        self.title = title
        self.columns = 2
        self.thumb_size = 400
        self.thumb_quality = DEFAULT_QUALITY
        self.medium_size = 1024
        self.medium_quality = DEFAULT_QUALITY
        self.bgcolor = "#FFFFFF"
        self.textcolor = "#000000"
        self.bordercolor = "#000000"
        if derived_from != None:
            for P in PROPS:
                if hasattr(derived_from, P) and P != 'title':
                    setattr(self, P, getattr(derived_from, P))

    
    def __set_thumb_size(self, x):
        self.__thumb_size = int(x)
    def __get_thumb_size(self):
        return self.__thumb_size
    thumb_size = property(__get_thumb_size, __set_thumb_size)
            
    def __set_thumb_quality(self, x):
        self.__thumb_quality = int(x)
    def __get_thumb_quality(self):
        return self.__thumb_quality
    thumb_quality = property(__get_thumb_quality, __set_thumb_quality)
        
    def __set_medium_size(self, x):
        self.__medium_size = int(x)
    def __get_medium_size(self):
        return self.__medium_size
    medium_size = property(__get_medium_size, __set_medium_size)
            
    def __set_medium_quality(self, x):
        self.__medium_quality = int(x)
    def __get_medium_quality(self):
        return self.__medium_quality
    medium_quality = property(__get_medium_quality, __set_medium_quality)
    
    def __set_columns(self, x):
        self.__columns = int(x)
    def __get_columns(self):
        return self.__columns
    columns = property(__get_columns, __set_columns)

    def __create_photo_dict(self):
        self.__photo_dict = {}
        for F in self.__photos:
            self.__photo_dict[F.name.lower()] = F

    def __set_title(self, title):
        title=str(title)
        self.__title = title.replace("\n", " ")
    def __get_title(self):
        return self.__title
    title = property(__get_title, __set_title)

    def sort(self):
        """
        Sort by filename.
        """
        self.__photos.sort()
        self.__create_photo_dict()

    def show(self):
        """
        Show all the photos in the album (using symlinks and the viewer).
        """
        DIR = ".view"
        if os.path.exists(DIR):
            if os.path.isdir(DIR):
                for f in os.listdir(DIR):
                    os.unlink(DIR + "/"+f)
            else:
                os.unlink(DIR)
                os.mkdir(DIR)                
        else:
            os.mkdir(DIR)
        for P in self:
            print P.filename
            os.symlink("../" + P.filename, DIR + "/" + P.name)
        os.system("cd .view; %s .&"%VIEWER)


    def __repr__(self):
        s="Album '%s' with %s photos:\n"%(self.title, len(self))
        for F in self:
            s += str(F) + "\n"
        return s

    def __len__(self):
        return len(self.__photos)

    def __getitem__(self, x):
        if isinstance(x, int):
            try:
                return self.__photos[x]
            except IndexError:
                raise IndexError, "The index must satisfy 0 <= i < %s"%len(self)
        else:
            try:
                return self.__photo_dict[str(x).lower()]
            except KeyError:
                raise KeyError, "No photo with name '%s'"%x


    def _get_photos(self):
        """
        Return list of photos.
        """
        return self.__photos
    photos = property(_get_photos)

    def photo(self, name):
        try:
            return self.__photo_dict[name]
        except KeyError:
            raise KeyError, "No photo named '%s' in the album."%name

    @prop
    def author_link(self):
        if hasattr(self, "author"):
            author = self.author
            if hasattr(self, "url"):
                return 'Photo by <A href="http://%s">%s</A>.'%(
                    self.url, author)
            elif hasattr(self, "email"):
                return 'Photo by <A href="mailto:%s">%s</A>.'%(
                    self.email, author)
            else:
                return 'Photo by %s.'%author
        return ""


    def html(self, name="html", delete_old_dir=False):
        print "Generating gallery %s"%self.title
        dir = os.curdir + "/" + name
        if os.path.exists(dir):
            if not delete_old_dir:
                raise IOError, "The directory '%s' already exists.  Please delete it first."%dir
            else:
                if os.path.isdir(dir):
                    for F in os.listdir(dir):
                        os.unlink(dir + "/" + F)
                else:
                    os.unlink(dir)
                    os.mkdir(dir)
        else:
            os.mkdir(dir)

        body = ""
        
        body += """
        <DIV align="center">
        <H1>%s</H1>
        """%self.title

        if hasattr(self, 'date'):
            body += '<H2>%s</H2>'%self.date

        body += '<P>%s</P>\n'%self.author_link.replace('Photo', 'Photos')

        body += '<TABLE cellpadding="8" border="0">\n\n'
        cols = self.columns
        thumb_size = self.thumb_size
        thumb_quality = self.thumb_quality
        medium_size = self.medium_size
        medium_quality = self.medium_quality
        for i in xrange(len(self)):
            P = self[i]
            print P
            if i % cols == 0:
                body += '<TR>\n'
            ext = P.ext
            j = name.rfind(".")
            base = P.base

            if P.is_photo:
                thumb_name = P.small(size=thumb_size, rotate=True, quality=thumb_quality)
                fname = '%s-thumb.jpg'%P.base
                os.symlink("../" + thumb_name, dir + "/" + fname)
                body += """
                   <TD width=%s align=center>
                   <A href="%s.html" alt="%s">
                   <IMG src="%s">
                   </A><BR>%s</TD>
                   """%(thumb_size+50, base, P.caption, fname, P.caption)
            else:  # It's a video
                body += """
                   <TD width=%s>
                   <A href="%s.html">
                   <TABLE width=%s bgcolor=%s cellpadding=8><tr><td align=center bgcolor=white>VIDEO CLIP</td></tr>  \
                    <tr><td align=center  bgcolor=white><A href="%s" alt="%s">%s</A></td></tr></table></A>
                    </TD>
                   """%(thumb_size+50, base, thumb_size+50, self.bordercolor, P.name, P.caption, P.name)
            #endif
            if (i+1)%cols == 0 or i == len(self)-1:
                body += '\n</TR>\n'
            i_prev = i - 1
            if i_prev < 0:
                i_prev = len(self)-1
            i_next = i + 1
            if i_next >= len(self):
                i_next = 0
            P.html(self, dir, self[i_prev], self[i_next], medium_size, medium_quality)

        body += '\n</TABLE>\n'
        body += '</DIV>\n\n'

        # Assemble
        s = """
        <HTML>
            <HEAD>
                <TITLE>
                %s
                </TITLE>
            </HEAD>
            <BODY bgcolor="%s">
            <FONT color="%s">
            %s
            </FONT>
            </BODY>
        </HTML>
        """%(self.title, self.bgcolor, self.textcolor, body)

        open("%s/index.html"%dir,"w").write(s)


    def conf(self, filename=None):
        s = "# Properties\n"
        for P in PROPS:
            if hasattr(self, P):
                s += "%s: %s\n"%(P, getattr(self, P))
        s += "\n# Photo ratings and captions\n"
        for x in self:
            s += str(x) + "\n"
        if filename==None:
            return s
        else:
            ext = extension(filename)
            if ext != "txt":
                filename += ".txt"
            open(filename,"w").write(s)

    def parse_conf(self, conf):
        """
        Parse the configuration file conf.  See the documentation for
        the album function (below) for more details.

        The parsing only applies to photos currently in this album.
        Information about any other photos is ignored.
        """
        c = open(conf).read()
        S = c.split("\n")
        c = "\n".join([x for x in S if len(x.lstrip()) and x.lstrip()[0] != "#"])
        while True:
            i = c.find(":")
            if i == -1:   # no further properties in the file
                return
            prop = c[:i]
            j = prop.rfind("\n")
            if j != -1:
                prop = prop[j+1:]
            ext = extension(prop).lower()
            c = c[i+1:].lstrip()
            if ext in EXTENSIONS:
                # photo or video
                rating = 0
                caption = ""
                # first get the rating:
                s = c.split()
                no_rating = False
                if len(s) > 0:
                    try:
                        rating = int(s[0])
                    except ValueError:
                        no_rating = True
                        pass
                    j = c.find(":")
                    if j == -1:
                        e = len(c)
                    else:
                        e = c[:j].rfind("\n")  
                    if no_rating:
                        j = -1
                    else:
                        j = c.find(s[0]) + len(s[0])
                    caption = c[j+1:e].lstrip().rstrip()
                    k = c[e:].find("\n")
                    c = c[e+k+1:]
                # set the rating and caption
                #print "Setting %s's rating to %s and caption to '%s'."%(
                #    prop, rating, caption)
                try:
                    F = self[prop]
                except KeyError:
                    print "WARNING: No photo '%s' in the album."%prop
                else:
                    F.rating = rating
                    F.caption = caption
                    
            else:  # property of the album
                j = c.find("\n")
                if j == -1:
                    j = len(c)
                value = c[:j].lstrip()
                c = c[j+1:]
                #print "Setting album property '%s' to '%s'."%(prop, value)
                prop = prop.lower()
                if not prop in PROPS:
                    print "WARNING: property %s will not be used."%prop
                setattr(self,prop,value)
        #end

    def __add__(self, other):
        if not isinstance(other, Album):
            raise TypeError, "other must be an Album."
        photos = self.photos + other.photos
        title = self.title + " + " + other.title
        return Album(photos, title, self)

    def save(self, filename):
        cPickle.dump(self, open(filename,"w"))

    def rated_atleast(self, r):
        r = int(r)
        title = self.title + " (rated at least %s)"%r
        photos = [x for x in self if x.rating >= r]
        return Album(photos, title, self)

    def rated_atmost(self, r):
        r = int(r)
        title = self.title + " (rated at most %s)"%r
        photos = [x for x in self if x.rating <= r]
        return Album(photos, title, self)

    def caption_contains(self, C):
        c = str(C).lower()
        title = self.title + " (caption contains %s)"%C
        photos = [x for x in self if x.caption.lower().find(c) != -1]
        return Album(photos, title, self)

    def caption_doesnt_contain(self, C):
        c = str(C).lower()
        title = self.title + " (caption does not contain %s)"%C
        photos = [x for x in self if x.caption.lower().find(c) == -1]
        return Album(photos, title, self)

    

def extension(file):
    i = file.rfind(".")
    if i == -1:
        return ""
    return file[i+1:]

def load_album(filename):
    return cPickle.load(open(filename))

def save_album(album, filename):
    album.save(filename)

def album(dir, conf=None):
    """
    Create an album from the files in a given directory.

    CAPTIONS, RATINGS, and FORMAT:

        If any text file (suffix .txt) is in the directory, this
        function extracts ratings, captions and other meta information
        from it.  The format of the file is the following:

           photo.jpg: 2 [This is a photo of a rainbow
           and a dog.]

           photo7.jpg: Tish William George

           video1.avi: 0 This is a video of a circus.

        The first entry is the name of one of the images (or videos)
        in the directory, followed by a colon.  Right after the name
        is a whole number, which is a rating for that, with bigger
        ratings being better.  The default rating is 0.  All text from
        there until the next image (or video, or end of file) is
        caption text.  The caption text is option and must be enclosed
        in square brackets if it spans more than one line.  The rating
        can also be optionally omitted, in which case it defaults to
        0.
        
        The txt file(s) can also be used to adjust various html
        parameters, the authors name, email address, etc.  The format
        is as follows:

            author: William Stein
            email: was@math.ucsd.edu
            bgcolor: white
            textcolor: darkgreen
            thumbsize: 200
            photosize: 800
            title: Trip to Hawaii in 2002

        The properties in the above example are all the available
        properties.

        For example, you could have one file named format.txt which
        includes general information, and the other file called
        captions.txt could contain just the photo captions.

    NOTES:
        * The caption text can be arbitrarily long and need not fit
          all on one line.  The caption text cannot contain brackets.

        * It's fine if there are entries in a txt file that don't
          correspond to actual files in the current directories.
          These are ignored.  This is allowed, since if you want to
          create a gallery from several directories, one way would be
          to copy some of the photos from each directory, then just
          concatenate the txt files.  That you have lots of extra
          entries in the notes.txt file won't cause a problem.

    INPUT:
        dir -- name of a directory
        
    OUTPUT:
        Album -- object of type album
    """
    if not os.path.isdir(dir):
        raise IOError, "No such directory: '%s'"%dir
    i = dir.rfind("/")
    if i != -1:
        title = dir[i+1:]
    else:
        title = dir
    photos = []
    for F in os.listdir(dir):
        ext = extension(F).lower()
        if ext in EXTENSIONS:
            P = Photo(dir + "/" + F, caption=F)
            photos.append(P)
    A = Album(photos, title)
    A.sort()
    
    # parse all the .txt files in the current directory.
    if conf == None:
        files = os.listdir(dir)
        files.sort()
        for F in files:
            ext = extension(F).lower()
            if ext == 'txt':
                A.parse_conf(dir + "/" + F)
    else:
        A.parse_conf(dir + "/" + conf)

    return A






###########################################################
# The exif package (which I didn't write)
# I'm including this here for simplicity of distribution.
###########################################################


###########################################################
# Library to extract EXIF information in digital camera image files
#
# Contains code from "exifdump.py" originally written by Thierry Bousch
# <bousch@topo.math.u-psud.fr> and released into the public domain.
#
# Updated and turned into general-purpose library by Gene Cash
# <gcash@cfl.rr.com>
#
# NOTE: This version has been modified by Leif Jensen
#
# This copyright license is intended to be similar to the FreeBSD license. 
#
# Copyright 2002 Gene Cash All rights reserved. 
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are
# met:
#
#    1. Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#    2. Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in the
#       documentation and/or other materials provided with the
#       distribution.
#
# THIS SOFTWARE IS PROVIDED BY GENE CASH ``AS IS'' AND ANY EXPRESS OR
# IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
# OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE FOR
# ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
# OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
# HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
# STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#
# This means you may do anything you want with this code, except claim you
# wrote it. Also, if it breaks you get to keep both pieces.
#
# 21-AUG-99 TB  Last update by Thierry Bousch to his code.
# 17-JAN-02 CEC Discovered code on web.
#               Commented everything.
#               Made small code improvements.
#               Reformatted for readability.
# 19-JAN-02 CEC Added ability to read TIFFs and JFIF-format JPEGs.
#               Added ability to extract JPEG formatted thumbnail.
#               Added ability to read GPS IFD (not tested).
#               Converted IFD data structure to dictionaries indexed by
#               tag name.
#               Factored into library returning dictionary of IFDs plus
#               thumbnail, if any.
# 20-JAN-02 CEC Added MakerNote processing logic.
#               Added Olympus MakerNote.
#               Converted data structure to single-level dictionary, avoiding
#               tag name collisions by prefixing with IFD name.  This makes
#               it much easier to use.
# 23-JAN-02 CEC Trimmed nulls from end of string values.
# 25-JAN-02 CEC Discovered JPEG thumbnail in Olympus TIFF MakerNote.
# 26-JAN-02 CEC Added ability to extract TIFF thumbnails.
#               Added Nikon, Fujifilm, Casio MakerNotes.
#
# To do:
# * Finish Canon MakerNote format
# * Better printing of ratios

# field type descriptions as (length, abbreviation, full name) tuples
FIELD_TYPES=(
    (0, 'X',  'Dummy'), # no such type
    (1, 'B',  'Byte'),
    (1, 'A',  'ASCII'),
    (2, 'S',  'Short'),
    (4, 'L',  'Long'),
    (8, 'R',  'Ratio'),
    (1, 'SB', 'Signed Byte'),
    (1, 'U',  'Undefined'),
    (2, 'SS', 'Signed Short'),
    (4, 'SL', 'Signed Long'),
    (8, 'SR', 'Signed Ratio')
    )

# dictionary of main EXIF tag names
# first element of tuple is tag name, optional second element is
# another dictionary giving names to values
EXIF_TAGS={
    0x0100: ('ImageWidth', ),
    0x0101: ('ImageLength', ),
    0x0102: ('BitsPerSample', ),
    0x0103: ('Compression',
             {1: 'Uncompressed TIFF',
              6: 'JPEG Compressed'}),
    0x0106: ('PhotometricInterpretation', ),
    0x010A: ('FillOrder', ),
    0x010D: ('DocumentName', ),
    0x010E: ('ImageDescription', ),
    0x010F: ('Make', ),
    0x0110: ('Model', ),
    0x0111: ('StripOffsets', ),
    0x0112: ('Orientation', ),
    0x0115: ('SamplesPerPixel', ),
    0x0116: ('RowsPerStrip', ),
    0x0117: ('StripByteCounts', ),
    0x011A: ('XResolution', ),
    0x011B: ('YResolution', ),
    0x011C: ('PlanarConfiguration', ),
    0x0128: ('ResolutionUnit',
             {1: 'Not Absolute',
              2: 'Pixels/Inch',
              3: 'Pixels/Centimeter'}),
    0x012D: ('TransferFunction', ),
    0x0131: ('Software', ),
    0x0132: ('DateTime', ),
    0x013B: ('Artist', ),
    0x013E: ('WhitePoint', ),
    0x013F: ('PrimaryChromaticities', ),
    0x0156: ('TransferRange', ),
    0x0200: ('JPEGProc', ),
    0x0201: ('JPEGInterchangeFormat', ),
    0x0202: ('JPEGInterchangeFormatLength', ),
    0x0211: ('YCbCrCoefficients', ),
    0x0212: ('YCbCrSubSampling', ),
    0x0213: ('YCbCrPositioning', ),
    0x0214: ('ReferenceBlackWhite', ),
    0x828D: ('CFARepeatPatternDim', ),
    0x828E: ('CFAPattern', ),
    0x828F: ('BatteryLevel', ),
    0x8298: ('Copyright', ),
    0x829A: ('ExposureTime', ),
    0x829D: ('FNumber', ),
    0x83BB: ('IPTC/NAA', ),
    0x8769: ('ExifOffset', ),
    0x8773: ('InterColorProfile', ),
    0x8822: ('ExposureProgram',
             {0: 'Unidentified',
              1: 'Manual',
              2: 'Program Normal',
              3: 'Aperture Priority',
              4: 'Shutter Priority',
              5: 'Program Creative',
              6: 'Program Action',
              7: 'Portrait Mode',
              8: 'Landscape Mode'}),
    0x8824: ('SpectralSensitivity', ),
    0x8825: ('GPSInfo', ),
    0x8827: ('ISOSpeedRatings', ),
    0x8828: ('OECF', ),
    0x9000: ('ExifVersion', ),
    0x9003: ('DateTimeOriginal', ),
    0x9004: ('DateTimeDigitized', ),
    0x9101: ('ComponentsConfiguration',
             {0: '',
              1: 'Y',
              2: 'Cb',
              3: 'Cr',
              4: 'Red',
              5: 'Green',
              6: 'Blue'}),
    0x9102: ('CompressedBitsPerPixel', ),
    0x9201: ('ShutterSpeedValue', ),
    0x9202: ('ApertureValue', ),
    0x9203: ('BrightnessValue', ),
    0x9204: ('ExposureBiasValue', ),
    0x9205: ('MaxApertureValue', ),
    0x9206: ('SubjectDistance', ),
    0x9207: ('MeteringMode',
             {0: 'Unidentified',
              1: 'Average',
              2: 'CenterWeightedAverage',
              3: 'Spot',
              4: 'MultiSpot'}),
    0x9208: ('LightSource',
             {0:   'Unknown',
              1:   'Daylight',
              2:   'Fluorescent',
              3:   'Tungsten',
              10:  'Flash',
              17:  'Standard Light A',
              18:  'Standard Light B',
              19:  'Standard Light C',
              20:  'D55',
              21:  'D65',
              22:  'D75',
              255: 'Other'}),
    0x9209: ('Flash', {0:  'No',
                       1:  'Fired',
                       5:  'Fired (?)', # no return sensed
                       7:  'Fired (!)', # return sensed
                       9:  'Fill Fired',
                       13: 'Fill Fired (?)',
                       15: 'Fill Fired (!)',
                       16: 'Off',
                       24: 'Auto Off',
                       25: 'Auto Fired',
                       29: 'Auto Fired (?)',
                       31: 'Auto Fired (!)',
                       32: 'Not Available'}),
    0x920A: ('FocalLength', ),
    0x927C: ('MakerNote', ),
    0x9286: ('UserComment', ),
    0x9290: ('SubSecTime', ),
    0x9291: ('SubSecTimeOriginal', ),
    0x9292: ('SubSecTimeDigitized', ),
    0xA000: ('FlashPixVersion', ),
    0xA001: ('ColorSpace', ),
    0xA002: ('ExifImageWidth', ),
    0xA003: ('ExifImageLength', ),
    0xA005: ('InteroperabilityOffset', ),
    0xA20B: ('FlashEnergy', ),               # 0x920B in TIFF/EP
    0xA20C: ('SpatialFrequencyResponse', ),  # 0x920C    -  -
    0xA20E: ('FocalPlaneXResolution', ),     # 0x920E    -  -
    0xA20F: ('FocalPlaneYResolution', ),     # 0x920F    -  -
    0xA210: ('FocalPlaneResolutionUnit', ),  # 0x9210    -  -
    0xA214: ('SubjectLocation', ),           # 0x9214    -  -
    0xA215: ('ExposureIndex', ),             # 0x9215    -  -
    0xA217: ('SensingMethod', ),             # 0x9217    -  -
    0xA300: ('FileSource',
             {3: 'Digital Camera'}),
    0xA301: ('SceneType',
             {1: 'Directly Photographed'}),
    }

# interoperability tags
INTR_TAGS={
    0x0001: ('InteroperabilityIndex', ),
    0x0002: ('InteroperabilityVersion', ),
    0x1000: ('RelatedImageFileFormat', ),
    0x1001: ('RelatedImageWidth', ),
    0x1002: ('RelatedImageLength', ),
    }

# GPS tags (not used yet, haven't seen camera with GPS)
GPS_TAGS={
    0x0000: ('GPSVersionID', ),
    0x0001: ('GPSLatitudeRef', ),
    0x0002: ('GPSLatitude', ),
    0x0003: ('GPSLongitudeRef', ),
    0x0004: ('GPSLongitude', ),
    0x0005: ('GPSAltitudeRef', ),
    0x0006: ('GPSAltitude', ),
    0x0007: ('GPSTimeStamp', ),
    0x0008: ('GPSSatellites', ),
    0x0009: ('GPSStatus', ),
    0x000A: ('GPSMeasureMode', ),
    0x000B: ('GPSDOP', ),
    0x000C: ('GPSSpeedRef', ),
    0x000D: ('GPSSpeed', ),
    0x000E: ('GPSTrackRef', ),
    0x000F: ('GPSTrack', ),
    0x0010: ('GPSImgDirectionRef', ),
    0x0011: ('GPSImgDirection', ),
    0x0012: ('GPSMapDatum', ),
    0x0013: ('GPSDestLatitudeRef', ),
    0x0014: ('GPSDestLatitude', ),
    0x0015: ('GPSDestLongitudeRef', ),
    0x0016: ('GPSDestLongitude', ),
    0x0017: ('GPSDestBearingRef', ),
    0x0018: ('GPSDestBearing', ),
    0x0019: ('GPSDestDistanceRef', ),
    0x001A: ('GPSDestDistance', )
    }

# Nikon E99x MakerNote Tags
# http://members.tripod.com/~tawba/990exif.htm
MAKERNOTE_NIKON_NEWER_TAGS={
    0x0002: ('ISOSetting', ),
    0x0003: ('ColorMode', ),
    0x0004: ('Quality', ),
    0x0005: ('Whitebalance', ),
    0x0006: ('ImageSharpening', ),
    0x0007: ('FocusMode', ),
    0x0008: ('FlashSetting', ),
    0x000F: ('ISOSelection', ),
    0x0080: ('ImageAdjustment', ),
    0x0082: ('AuxiliaryLens', ),
    0x0085: ('ManualFocusDistance', ),
    0x0086: ('DigitalZoomFactor', ),
    0x0088: ('AFFocusPosition',
             {0x0000: 'Center',
              0x0100: 'Top',
              0x0200: 'Bottom',
              0x0300: 'Left',
              0x0400: 'Right'}),
    0x0094: ('Saturation',
             {-3: 'B&W',
              -2: '-2',
              -1: '-1',
              0:  '0',
              1:  '1',
              2:  '2'}),
    0x0095: ('NoiseReduction', ),
    0x0010: ('DataDump', )
    }

MAKERNOTE_NIKON_OLDER_TAGS={
    0x0003: ('Quality',
             {1: 'VGA Basic',
              2: 'VGA Normal',
              3: 'VGA Fine',
              4: 'SXGA Basic',
              5: 'SXGA Normal',
              6: 'SXGA Fine'}),
    0x0004: ('ColorMode',
             {1: 'Color',
              2: 'Monochrome'}),
    0x0005: ('ImageAdjustment',
             {0: 'Normal',
              1: 'Bright+',
              2: 'Bright-',
              3: 'Contrast+',
              4: 'Contrast-'}),
    0x0006: ('CCDSpeed',
             {0: 'ISO 80',
              2: 'ISO 160',
              4: 'ISO 320',
              5: 'ISO 100'}),
    0x0007: ('WhiteBalance',
             {0: 'Auto',
              1: 'Preset',
              2: 'Daylight',
              3: 'Incandescent',
              4: 'Fluorescent',
              5: 'Cloudy',
              6: 'Speed Light'})
    }

# decode Olympus SpecialMode tag in MakerNote
def olympus_special_mode(v):
    a={
        0: 'Normal',
        1: 'Unknown',
        2: 'Fast',
        3: 'Panorama'}
    b={
        0: 'Non-panoramic',
        1: 'Left to right',
        2: 'Right to left',
        3: 'Bottom to top',
        4: 'Top to bottom'}
    return '%s - sequence %d - %s' % (a[v[0]], v[1], b[v[2]])
        
MAKERNOTE_OLYMPUS_TAGS={
    # ah HAH! those sneeeeeaky bastids! this is how they get past the fact
    # that a JPEG thumbnail is not allowed in an uncompressed TIFF file
    0x0100: ('JPEGThumbnail', ),
    0x0200: ('SpecialMode', olympus_special_mode),
    0x0201: ('JPEGQual',
             {1: 'SQ',
              2: 'HQ',
              3: 'SHQ'}),
    0x0202: ('Macro',
             {0: 'Normal',
              1: 'Macro'}),
    0x0204: ('DigitalZoom', ),
    0x0207: ('SoftwareRelease',  ),
    0x0208: ('PictureInfo',  ),
    # print as string
    0x0209: ('CameraID', lambda x: ''.join(map(chr, x))), 
    0x0F00: ('DataDump',  )
    }

MAKERNOTE_CASIO_TAGS={
    0x0001: ('RecordingMode',
             {1: 'Single Shutter',
              2: 'Panorama',
              3: 'Night Scene',
              4: 'Portrait',
              5: 'Landscape'}),
    0x0002: ('Quality',
             {1: 'Economy',
              2: 'Normal',
              3: 'Fine'}),
    0x0003: ('FocusingMode',
             {2: 'Macro',
              3: 'Auto Focus',
              4: 'Manual Focus',
              5: 'Infinity'}),
    0x0004: ('FlashMode',
             {1: 'Auto',
              2: 'On',
              3: 'Off',
              4: 'Red Eye Reduction'}),
    0x0005: ('FlashIntensity',
             {11: 'Weak',
              13: 'Normal',
              15: 'Strong'}),
    0x0006: ('Object Distance', ),
    0x0007: ('WhiteBalance',
             {1:   'Auto',
              2:   'Tungsten',
              3:   'Daylight',
              4:   'Fluorescent',
              5:   'Shade',
              129: 'Manual'}),
    0x000B: ('Sharpness',
             {0: 'Normal',
              1: 'Soft',
              2: 'Hard'}),
    0x000C: ('Contrast',
             {0: 'Normal',
              1: 'Low',
              2: 'High'}),
    0x000D: ('Saturation',
             {0: 'Normal',
              1: 'Low',
              2: 'High'}),
    0x0014: ('CCDSpeed',
             {64:  'Normal',
              80:  'Normal',
              100: 'High',
              125: '+1.0',
              244: '+3.0',
              250: '+2.0',})
    }

MAKERNOTE_FUJIFILM_TAGS={
    0x0000: ('NoteVersion', lambda x: ''.join(map(chr, x))),
    0x1000: ('Quality', ),
    0x1001: ('Sharpness',
             {1: 'Soft',
              2: 'Soft',
              3: 'Normal',
              4: 'Hard',
              5: 'Hard'}),
    0x1002: ('WhiteBalance',
             {0:    'Auto',
              256:  'Daylight',
              512:  'Cloudy',
              768:  'DaylightColor-Fluorescent',
              769:  'DaywhiteColor-Fluorescent',
              770:  'White-Fluorescent',
              1024: 'Incandescent',
              3840: 'Custom'}),
    0x1003: ('Color',
             {0:   'Normal',
              256: 'High',
              512: 'Low'}),
    0x1004: ('Tone',
             {0:   'Normal',
              256: 'High',
              512: 'Low'}),
    0x1010: ('FlashMode',
             {0: 'Auto',
              1: 'On',
              2: 'Off',
              3: 'Red Eye Reduction'}),
    0x1011: ('FlashStrength', ),
    0x1020: ('Macro',
             {0: 'Off',
              1: 'On'}),
    0x1021: ('FocusMode',
             {0: 'Auto',
              1: 'Manual'}),
    0x1030: ('SlowSync',
             {0: 'Off',
              1: 'On'}),
    0x1031: ('PictureMode',
             {0:   'Auto',
              1:   'Portrait',
              2:   'Landscape',
              4:   'Sports',
              5:   'Night',
              6:   'Program AE',
              256: 'Aperture Priority AE',
              512: 'Shutter Priority AE',
              768: 'Manual Exposure'}),
    0x1100: ('MotorOrBracket',
             {0: 'Off',
              1: 'On'}),
    0x1300: ('BlurWarning',
             {0: 'Off',
              1: 'On'}),
    0x1301: ('FocusWarning',
             {0: 'Off',
              1: 'On'}),
    0x1302: ('AEWarning',
             {0: 'Off',
              1: 'On'})
    }

MAKERNOTE_CANON_TAGS={
    0x0006: ('ImageType', ),
    0x0007: ('FirmwareVersion', ),
    0x0008: ('ImageNumber', ),
    0x0009: ('OwnerName', )
    }

# see http://www.burren.cx/david/canon.html by David Burren
# this is in element offset, name, optional value dictionary format
MAKERNOTE_CANON_TAG_0x001={
    1: ('Macromode',
        {1: 'Macro',
         2: 'Normal'}),
    2: ('SelfTimer', ),
    3: ('Quality',
        {2: 'Normal',
         3: 'Fine',
         5: 'Superfine'}),
    4: ('FlashMode',
        {0: 'Flash Not Fired',
         1: 'Auto',
         2: 'On',
         3: 'Red-Eye Reduction',
         4: 'Slow Synchro',
         5: 'Auto + Red-Eye Reduction',
         6: 'On + Red-Eye Reduction',
         16: 'external flash'}),
    5: ('ContinuousDriveMode',
        {0: 'Single Or Timer',
         1: 'Continuous'}),
    7: ('FocusMode',
        {0: 'One-Shot',
         1: 'AI Servo',
         2: 'AI Focus',
         3: 'MF',
         4: 'Single',
         5: 'Continuous',
         6: 'MF'}),
    10: ('ImageSize',
         {0: 'Large',
          1: 'Medium',
          2: 'Small'}),
    11: ('EasyShootingMode',
         {0: 'Full Auto',
          1: 'Manual',
          2: 'Landscape',
          3: 'Fast Shutter',
          4: 'Slow Shutter',
          5: 'Night',
          6: 'B&W',
          7: 'Sepia',
          8: 'Portrait',
          9: 'Sports',
          10: 'Macro/Close-Up',
          11: 'Pan Focus'}),
    12: ('DigitalZoom',
         {0: 'None',
          1: '2x',
          2: '4x'}),
    13: ('Contrast',
         {0xFFFF: 'Low',
          0: 'Normal',
          1: 'High'}),
    14: ('Saturation',
         {0xFFFF: 'Low',
          0: 'Normal',
          1: 'High'}),
    15: ('Sharpness',
         {0xFFFF: 'Low',
          0: 'Normal',
          1: 'High'}),
    16: ('ISO',
         {0: 'See ISOSpeedRatings Tag',
          15: 'Auto',
          16: '50',
          17: '100',
          18: '200',
          19: '400'}),
    17: ('MeteringMode',
         {3: 'Evaluative',
          4: 'Partial',
          5: 'Center-weighted'}),
    18: ('FocusType',
         {0: 'Manual',
          1: 'Auto',
          3: 'Close-Up (Macro)',
          8: 'Locked (Pan Mode)'}),
    19: ('AFPointSelected',
         {0x3000: 'None (MF)',
          0x3001: 'Auto-Selected',
          0x3002: 'Right',
          0x3003: 'Center',
          0x3004: 'Left'}),
    20: ('ExposureMode',
         {0: 'Easy Shooting',
          1: 'Program',
          2: 'Tv-priority',
          3: 'Av-priority',
          4: 'Manual',
          5: 'A-DEP'}),
    23: ('LongFocalLengthOfLensInFocalUnits', ),
    24: ('ShortFocalLengthOfLensInFocalUnits', ),
    25: ('FocalUnitsPerMM', ),
    28: ('FlashActivity',
         {0: 'Did Not Fire',
          1: 'Fired'}),
    29: ('FlashDetails',
         {14: 'External E-TTL',
          13: 'Internal Flash',
          11: 'FP Sync Used',
          7: '2nd("Rear")-Curtain Sync Used',
          4: 'FP Sync Enabled'}),
    32: ('FocusMode',
         {0: 'Single',
          1: 'Continuous'})
    }

MAKERNOTE_CANON_TAG_0x004={
    7: ('WhiteBalance',
        {0: 'Auto',
         1: 'Sunny',
         2: 'Cloudy',
         3: 'Tungsten',
         4: 'Fluorescent',
         5: 'Flash',
         6: 'Custom'}),
    9: ('SequenceNumber', ),
    14: ('AFPointUsed', ),
    15: ('FlashBias',
        {0XFFC0: '-2 EV',
         0XFFCC: '-1.67 EV',
         0XFFD0: '-1.50 EV',
         0XFFD4: '-1.33 EV',
         0XFFE0: '-1 EV',
         0XFFEC: '-0.67 EV',
         0XFFF0: '-0.50 EV',
         0XFFF4: '-0.33 EV',
         0X0000: '0 EV',
         0X000C: '0.33 EV',
         0X0010: '0.50 EV',
         0X0014: '0.67 EV',
         0X0020: '1 EV',
         0X002C: '1.33 EV',
         0X0030: '1.50 EV',
         0X0034: '1.67 EV',
         0X0040: '2 EV'}), 
    19: ('SubjectDistance', )
    }

# extract multibyte integer in Motorola format (little endian)
def s2n_motorola(str):
    x=0
    for c in str:
        x=(long(x) << 8) | ord(c)
    return x

# extract multibyte integer in Intel format (big endian)
def s2n_intel(str):
    x=0
    y=0
    for c in str:
        x= x | (long(ord(c)) << y)
        y=y+8
    return x

# ratio object that eventually will be able to reduce itself to lowest
# common denominator for printing
def gcd(a, b):
   if b == 0:
      return a
   else:
      return gcd(b, a % b)

class Ratio:
    def __init__(self, num, den):
        self.num=num
        self.den=den

    def __repr__(self):
#       self.reduce() # ugh, 259/250 worse 1036/1000
        if self.den == 1:
            return str(self.num)
        return '%d/%d' % (self.num, self.den)

    def reduce(self):
        div=gcd(self.num, self.den)
        if div > 1:
            self.num=self.num/div
            self.den=self.den/div

# for ease of dealing with tags
class IFD_Tag:
    def __init__(self, printable, tag, field_type, values, field_offset,
                 field_length):
        self.printable=printable
        self.tag=tag
        self.field_type=field_type
        self.field_offset=field_offset
        self.field_length=field_length
        self.values=values
        
    def __str__(self):
        return self.printable
    
    def __repr__(self):
        return '(0x%04X) %s=%s @ %d' % (self.tag,
                                        FIELD_TYPES[self.field_type][2],
                                        self.printable,
                                        self.field_offset)

# class that handles an EXIF header
class EXIF_header:
    def __init__(self, file, endian, offset, debug=0):
        self.file=file
        self.endian=endian
        self.offset=offset
        self.debug=debug
        self.tags={}
        
    # convert slice to integer, based on sign and endian flags
    def s2n(self, offset, length, signed=0):
        self.file.seek(self.offset+offset)
        slice=self.file.read(length)
        if self.endian == 'I':
            val=s2n_intel(slice)
        else:
            val=s2n_motorola(slice)
        # Sign extension ?
        if signed:
            msb = long(1) << (8*length-1)
            if val & msb:
                val=val-(msb << 1)
        return val

    # convert offset to string
    def n2s(self, offset, length):
        s=''
        for i in range(length):
            if self.endian == 'I':
                s=s+chr(offset & 0xFF)
            else:
                s=chr(offset & 0xFF)+s
            offset=offset >> 8
        return s
    
    # return first IFD
    def first_IFD(self):
        return self.s2n(4, 4)

    # return pointer to next IFD
    def next_IFD(self, ifd):
        entries=self.s2n(ifd, 2)
        return self.s2n(ifd+2+12*entries, 4)

    # return list of IFDs in header
    def list_IFDs(self):
        i=self.first_IFD()
        a=[]
        while i:
            a.append(i)
            i=self.next_IFD(i)
        return a

    # return list of entries in this IFD
    def dump_IFD(self, ifd, ifd_name, dict=EXIF_TAGS):
        entries=self.s2n(ifd, 2)
        for i in range(entries):
            entry=ifd+2+12*i
            tag=self.s2n(entry, 2)
            field_type=self.s2n(entry+2, 2)
            if not 0 < field_type < len(FIELD_TYPES):
                # unknown field type
                raise ValueError, \
                      'unknown type %d in tag 0x%04X' % (field_type, tag)
            typelen=FIELD_TYPES[field_type][0]
            count=self.s2n(entry+4, 4)
            offset=entry+8
            if count*typelen > 4:
                # not the value, it's a pointer to the value
                offset=self.s2n(offset, 4)
            field_offset=offset
            if field_type == 2:
                # special case: null-terminated ASCII string
                if count != 0:
                    self.file.seek(self.offset+offset)
                    values=self.file.read(count).strip().replace('\x00','')
                else:
                    values=''
            elif tag == 0x927C or tag == 0x9286: # MakerNote or UserComment
#            elif tag == 0x9286: # UserComment
                values=[]
            else:
                values=[]
                signed=(field_type in [6, 8, 9, 10])
                for j in range(count):
                    if field_type in (5, 10):
                        # a ratio
                        value_j=Ratio(self.s2n(offset,   4, signed),
                                      self.s2n(offset+4, 4, signed))
                    else:
                        value_j=self.s2n(offset, typelen, signed)
                    values.append(value_j)
                    offset=offset+typelen
            # now "values" is either a string or an array
            if count == 1 and field_type != 2:
                printable=str(values[0])
            else:
                printable=str(values)
            # figure out tag name
            tag_entry=dict.get(tag)
            if tag_entry:
                tag_name=tag_entry[0]
                if len(tag_entry) != 1:
                    # optional 2nd tag element is present
                    if callable(tag_entry[1]):
                        # call mapping function
                        printable=tag_entry[1](values)
                    else:
                        printable=''
                        for i in values:
                            # use LUT for this tag
                            printable+=tag_entry[1].get(i, repr(i))
            else:
                tag_name='Tag 0x%04X' % tag
            self.tags[ifd_name+' '+tag_name]=IFD_Tag(printable, tag,
                                                     field_type,
                                                     values, field_offset,
                                                     count*typelen)
            if self.debug:
                print '    %s: %s' % (tag_name,
                                      repr(self.tags[ifd_name+' '+tag_name]))

    # extract uncompressed TIFF thumbnail (like pulling teeth)
    # we take advantage of the pre-existing layout in the thumbnail IFD as
    # much as possible
    def extract_TIFF_thumbnail(self, thumb_ifd):
        entries=self.s2n(thumb_ifd, 2)
        # this is header plus offset to IFD ...
        if self.endian == 'M':
            tiff='MM\x00*\x00\x00\x00\x08'
        else:
            tiff='II*\x00\x08\x00\x00\x00'
        # ... plus thumbnail IFD data plus a null "next IFD" pointer
        self.file.seek(self.offset+thumb_ifd)
        tiff+=self.file.read(entries*12+2)+'\x00\x00\x00\x00'
        
        # fix up large value offset pointers into data area
        for i in range(entries):
            entry=thumb_ifd+2+12*i
            tag=self.s2n(entry, 2)
            field_type=self.s2n(entry+2, 2)
            typelen=FIELD_TYPES[field_type][0]
            count=self.s2n(entry+4, 4)
            oldoff=self.s2n(entry+8, 4)
            # start of the 4-byte pointer area in entry
            ptr=i*12+18
            # remember strip offsets location
            if tag == 0x0111:
                strip_off=ptr
                strip_len=count*typelen
            # is it in the data area?
            if count*typelen > 4:
                # update offset pointer (nasty "strings are immutable" crap)
                # should be able to say "tiff[ptr:ptr+4]=newoff"
                newoff=len(tiff)
                tiff=tiff[:ptr]+self.n2s(newoff, 4)+tiff[ptr+4:]
                # remember strip offsets location
                if tag == 0x0111:
                    strip_off=newoff
                    strip_len=4
                # get original data and store it
                self.file.seek(self.offset+oldoff)
                tiff+=self.file.read(count*typelen)
                
        # add pixel strips and update strip offset info
        old_offsets=self.tags['Thumbnail StripOffsets'].values
        old_counts=self.tags['Thumbnail StripByteCounts'].values
        for i in range(len(old_offsets)):
            # update offset pointer (more nasty "strings are immutable" crap)
            offset=self.n2s(len(tiff), strip_len)
            tiff=tiff[:strip_off]+offset+tiff[strip_off+strip_len:]
            strip_off+=strip_len
            # add pixel strip to end
            self.file.seek(self.offset+old_offsets[i])
            tiff+=self.file.read(old_counts[i])
            
        self.tags['TIFFThumbnail']=tiff
        
    # decode all the camera-specific MakerNote formats
    def decode_maker_note(self):
        note=self.tags['EXIF MakerNote']
        make=self.tags['Image Make'].printable
        model=self.tags['Image Model'].printable

        # Nikon
        if make == 'NIKON':
            if note.values[0:5] == [78, 105, 107, 111, 110]: # "Nikon"
                # older model
                self.dump_IFD(note.field_offset+8, 'MakerNote',
                              dict=MAKERNOTE_NIKON_OLDER_TAGS)
            else:
                # newer model (E99x or D1)
                self.dump_IFD(note.field_offset, 'MakerNote',
                              dict=MAKERNOTE_NIKON_NEWER_TAGS)
            return

        # Olympus
        if make[:7] == 'OLYMPUS':
            self.dump_IFD(note.field_offset+8, 'MakerNote',
                          dict=MAKERNOTE_OLYMPUS_TAGS)
            return

        # Casio
        if make == 'Casio':
            self.dump_IFD(note.field_offset, 'MakerNote',
                          dict=MAKERNOTE_CASIO_TAGS)
            return
        
        # Fujifilm
        if make == 'FUJIFILM':
            # bug: everything else is "Motorola" endian, but the MakerNote
            # is "Intel" endian 
            endian=self.endian
            self.endian='I'
            # bug: IFD offsets are from beginning of MakerNote, not
            # beginning of file header
            offset=self.offset
            self.offset+=note.field_offset
            # process note with bogus values (note is actually at offset 12)
            self.dump_IFD(12, 'MakerNote', dict=MAKERNOTE_FUJIFILM_TAGS)
            # reset to correct values
            self.endian=endian
            self.offset=offset
            return
        
        # Canon
        if make == 'Canon':
            self.dump_IFD(note.field_offset, 'MakerNote',
                          dict=MAKERNOTE_CANON_TAGS)
            for i in (('MakerNote Tag 0x0001', MAKERNOTE_CANON_TAG_0x001),
                      ('MakerNote Tag 0x0004', MAKERNOTE_CANON_TAG_0x004)):
                if self.debug:
                  print ' SubMakerNote BitSet for ' +i[0]
                self.canon_decode_tag(self.tags[i[0]].values, i[1])
            return

    # decode Canon MakerNote tag based on offset within tag
    # see http://www.burren.cx/david/canon.html by David Burren
    def canon_decode_tag(self, value, dict):
        for i in range(1, len(value)):
            x=dict.get(i, ('Unknown', ))
#            if self.debug:
#                print i, x
            name=x[0]
            if len(x) > 1:
                val=x[1].get(value[i], 'Unknown')
            else:
                val=value[i]
            if self.debug:
                print '      '+name+':', val
            self.tags['MakerNote '+name]=val

# process an image file (expects an open file object)
# this is the function that has to deal with all the arbitrary nasty bits
# of the EXIF standard
def process_file(file, debug=0, noclose=0):
    # determine whether it's a JPEG or TIFF
    data=file.read(12)
    if data[0:4] in ['II*\x00', 'MM\x00*']:
        # it's a TIFF file
        file.seek(0)
        endian=file.read(1)
        file.read(1)
        offset=0
    elif data[0:2] == '\xFF\xD8':
        # it's a JPEG file
        # skip JFIF style header(s)
        while data[2] == '\xFF' and data[6:10] in ('JFIF', 'JFXX', 'OLYM'):
            length=ord(data[4])*256+ord(data[5])
            file.read(length-8)
            # fake an EXIF beginning of file
            data='\xFF\x00'+file.read(10)
        if data[2] == '\xFF' and data[6:10] == 'Exif':
            # detected EXIF header
            offset=file.tell()
            endian=file.read(1)
        else:
            # no EXIF information
            return {}
    else:
        # file format not recognized
        return {}

    # deal with the EXIF info we found
    if debug:
        print {'I': 'Intel', 'M': 'Motorola'}[endian], 'format'
    hdr=EXIF_header(file, endian, offset, debug)
    ifd_list=hdr.list_IFDs()
    ctr=0
    for i in ifd_list:
        if ctr == 0:
            IFD_name='Image'
        elif ctr == 1:
            IFD_name='Thumbnail'
            thumb_ifd=i
        else:
            IFD_name='IFD %d' % ctr
        if debug:
            print ' IFD %d (%s) at offset %d:' % (ctr, IFD_name, i)
        hdr.tags['Exif Offset'] = offset
        hdr.tags[IFD_name+' IFDOffset'] = i
        hdr.dump_IFD(i, IFD_name)
        # EXIF IFD
        exif_off=hdr.tags.get(IFD_name+' ExifOffset')
        if exif_off:
            if debug:
                print ' EXIF SubIFD at offset %d:' % exif_off.values[0]
            hdr.dump_IFD(exif_off.values[0], 'EXIF')
            # Interoperability IFD contained in EXIF IFD
            #intr_off=hdr.tags.get('EXIF SubIFD InteroperabilityOffset')
            intr_off=hdr.tags.get('EXIF InteroperabilityOffset')
            if intr_off:
                if debug:
                    print ' EXIF Interoperability SubSubIFD at offset %d:' \
                          % intr_off.values[0]
                hdr.dump_IFD(intr_off.values[0], 'EXIF Interoperability',
                             dict=INTR_TAGS)
            # deal with MakerNote contained in EXIF IFD
            if hdr.tags.has_key('EXIF MakerNote'):
                if debug:
                    print ' EXIF MakerNote SubSubIFD at offset %d:' \
                          % intr_off.values[0]
                hdr.decode_maker_note()
        # GPS IFD
        gps_off=hdr.tags.get(IFD_name+' GPSInfoOffset')
        if gps_off:
            if debug:
                print ' GPS SubIFD at offset %d:' % gps_off.values[0]
            hdr.dump_IFD(gps_off.values[0], 'GPS', dict=GPS_TAGS)
        ctr+=1


    # extract uncompressed TIFF thumbnail
    thumb=hdr.tags.get('Thumbnail Compression')
    if thumb and thumb.printable == 'Uncompressed TIFF':
        hdr.extract_TIFF_thumbnail(thumb_ifd)
        
    # JPEG thumbnail (thankfully the JPEG data is stored as a unit)
    thumb_off=hdr.tags.get('Thumbnail JPEGInterchangeFormat')
    if thumb_off:
        file.seek(offset+thumb_off.values[0])
        size=hdr.tags['Thumbnail JPEGInterchangeFormatLength'].values[0]
        hdr.tags['JPEGThumbnail']=file.read(size)
        
    # Sometimes in a TIFF file, a JPEG thumbnail is hidden in the MakerNote
    # since it's not allowed in a uncompressed TIFF IFD
    if not hdr.tags.has_key('JPEGThumbnail'):
        thumb_off=hdr.tags.get('MakerNote JPEGThumbnail')
        if thumb_off:
            file.seek(offset+thumb_off.values[0])
            hdr.tags['JPEGThumbnail']=file.read(thumb_off.field_length)
            
    if noclose == 0:
      file.close()
    return hdr.tags




##########################################################################


###############################################################
# What to do with album.py, when run as a shell script
###############################################################


if __name__ ==  '__main__':
    dir = ".html"
    import sys
    argv = sys.argv
    if len(argv) == 1:
        prog = os.path.split(argv[0])[1]
        #argv.append("0")
        s =  "****************************************************************************\n"
        s += "* HTML Gallery Program Version 1.0                                         *\n"
        s += "* Copyright (C) 2005 William Stein <was@math.harvard.edu>                  *\n"
        s += "* Distributed under the terms of the GNU General Public License (GPL)      *\n"
        s += "****************************************************************************\n"
        s += "\n"
        s += "Usage: %s min_rating [+keyword] [-keyword] ...\n"%(prog)
        s += "\n"
        s += "---------------------------------------------------------------------------\n"
	s +=  " Create an HTML gallery of photos with rating at least min_rating,\n"
	s +=  " making use of any .txt files in the current directory for rating,\n"
        s +=  " captioning, and style information.  The photos are sorted by date/time.\n"
        s +=  "\n"
	s +=  " The default photo rating is 0, so type '%s 0' to make \n"%prog
	s +=  " a gallery of all photos (except any you have rated negative).\n"
	s +=  " If +keyword appears for any keyword, only photos whose caption\n"
	s +=  " contains keyword will be included.  You can include any number\n"
	s +=  " of +keywords, to include only photos that contain *all* those words.\n"
	s +=  " Likewise, -keyword only includes photos that do not contain keyword.\n"
	s +=  " The keywords are not case sensitive.\n"
	s +=  "\n"
	s +=  "EXAMPLES:\n"
	s +=  " * Make gallery of all photos in current directory not rated negative.\n"
	s +=  "       %s 0\n"%prog
	s +=  "\n"
	s +=  " * Create gallery of photos rated at least 2 that don't contain\n"
	s +=  "   the word Kelly but do contain William.\n"
	s +=  "       %s 2 -kelly +william\n"%prog
	s +=  "\n"
        s +=  "RATINGS AND CAPTIONS:\n"
	s +=  " The %s program makes use of any file with extension .txt in the\n"%prog
        s +=  " current directory, parsing all of them in alphabetical order.\n"
        s +=  " The format of a .txt file is a series of lines as follows:\n"
        s +=  "\n"
	s +=  "     property: value\n"
        s +=  "     image_name: rating caption\n"
        s +=  "     # a comment begins with # and will be ignored.\n"
        s +=  "\n"
 	s +=  " * image_name must be the file name of an image (or video) including\n "
        s +=  "   the extension, e.g., img_7430.jpg.  The rating must be an integer,\n"
        s +=  "   and the caption text is any text that does not contain a colon.\n"
        s +=  "   The rating may be omitted, and the caption text may span multiple lines.\n"
        s +=  " * The properties are: " + ', '.join(PROPS) + "\n"
        s +=  "\n"
        s +=  " The thumbnails, small versions of photos, HTML gallery, and exif info \n"
        s +=  " are stored in hidden subdirectories .exif, .small, and .html.\n"  
        s +=  "---------------------------------------------------------------------------\n"
        s += "\n"
        open(".tmp_album","w").write(s)
        os.system("less .tmp_album")
        os.unlink(".tmp_album")

        sys.exit(0)
        
    a = album('.')
    if a.title == ".":
        a.title = os.path.split(os.path.abspath('.'))[1]
    t = a.title
    if len(argv) > 1:
        min_rating = int(argv[1])
        a = a.rated_atleast(min_rating)
    if len(argv) > 2:
        for v in argv[2:]:
            if v[0] != "+" and v[0] != "-":
                v = "+" + v
            if v[0] == "+":
                a = a.caption_contains(v[1:])
            elif v[0] == "-":
                a = a.caption_doesnt_contain(v[1:])
    a.title = t
    a.sort()  
    a.html(dir, delete_old_dir=True)
    index = open("%s/index.html"%dir).read()
    i = index.find("TABLE")
    x = index[i:]
    x = x.replace('href="', 'href="%s/'%dir)
    x = x.replace('src="', 'src="%s/'%dir)
    index = index[:i] + x
    open("index.html","w").write(index)
