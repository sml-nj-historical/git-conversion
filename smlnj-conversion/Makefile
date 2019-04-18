# Makefile for smlnj conversion using reposurgeon
#
# Steps to using this:
# 1. Make sure reposurgeon and repotool are on your $PATH.
#    For large repositories It is usually best to run reposurgeon using
#    PyPy: set REPOSURGEON to "pypy" followed by an absolute pathname
#    to reposurgeon.
# 2. (Skip this step if you're starting from a stream file.) For svn, set
#    REMOTE_URL to point at the remote repository you want to convert.
#    If the repository is already in a DVCS such as hg or git,
#    set REMOTE_URL to either the normal cloning URL (starting with hg://,
#    git://, etc.) or to the path of a local clone.
# 3. For cvs, set CVS_HOST to the repo hostname and CVS_MODULE to the module,
#    then uncomment the line that builds REMOTE_URL
#    Note: for CVS hosts other than Sourceforge or Savannah you will need to
#    include the path to the CVS modules directory after the hostname.
# 4. Set any required read options, such as --user-ignores or --nobranch,
#    by setting READ_OPTIONS.
# 5. Run 'make stubmap' to create a stub author map.
# 6. (Optional) set REPOSURGEON to point at a faster cython build of the tool.
# 7. Run 'make' to build a converted repository.
#
# The reason both first- and second-stage stream files are generated is that,
# especially with Subversion, making the first-stage stream file is often
# painfully slow. By splitting the process, we lower the overhead of
# experiments with the lift script.
#
# For a production-quality conversion you will need to edit the map
# file and the lift script.  During the process you can set EXTRAS to
# name extra metadata such as a comments mailbox.
#
# Afterwards, you can use the headcompare and tagscompare productions
# to check your work.
#

EXTRAS =
REMOTE_URL = svn://svn.debian.org/smlnj # unused
#REMOTE_URL = https://smlnj.googlecode.com/svn/
# CVS_HOST = smlnj.cvs.sourceforge.net
#CVS_HOST = cvs.savannah.gnu.org
# CVS_MODULE = smlnj
#REMOTE_URL = cvs://$(CVS_HOST)/smlnj\#$(CVS_MODULE)
READ_OPTIONS =
VERBOSITY = "verbose 1"
REPOSURGEON = reposurgeon

# Configuration ends here

.PHONY: local-clobber remote-clobber gitk gc compare clean dist stubmap
# Tell make not to auto-remove tag directories, because it only tries rm
# and hence fails
.PRECIOUS: smlnj-%-checkout smlnj-%-git

default: smlnj-git

# Build the converted repo from the second-stage fast-import stream
smlnj-git: smlnj.fi
	rm -fr smlnj-git; $(REPOSURGEON) 'read <smlnj.fi' 'prefer git' 'rebuild smlnj-git'

# Build the second-stage fast-import stream from the first-stage stream dump
smlnj.fi: smlnj.opts smlnj.lift smlnj.map $(EXTRAS)
	$(REPOSURGEON) $(VERBOSITY) 'script smlnj.opts' "read $(READ_OPTIONS) <smlnj.svn" 'authors read <smlnj.map' 'sourcetype svn' 'prefer git' 'script smlnj.lift' 'legacy write >smlnj.fo' 'write >smlnj.fi'

# Build the first-stage stream dump from the local mirror
## The repocutter steps need to be done manually on a shell as its backreferences
## support is messed up.
smlnj.svn:
	(cd smlnj-mirror/ >/dev/null; repotool export) \
	| repocutter pathrename '^trunk/' '' \
	| repocutter swap \
	| repocutter pathrename '^branches/([^/]+)/([^/]+)/' 'branches/${2}/${1}/' \
	| repocutter pathrename '^releases/([^/]+)/([^/]+)/' 'tags/${2}/${1}/' \
	>smlnj.svn

# Build a local mirror of the remote repository
smlnj-mirror:
	repotool mirror $(REMOTE_URL) smlnj-mirror

# Make a local checkout of the source mirror for inspection
smlnj-checkout: smlnj-mirror
	cd smlnj-mirror >/dev/null; repotool checkout $(PWD)/smlnj-checkout

# Make a local checkout of the source mirror for inspection at a specific tag
smlnj-%-checkout: smlnj-mirror
	cd smlnj-mirror >/dev/null; repotool checkout $(PWD)/smlnj-$*-checkout $*

# Force rebuild of first-stage stream from the local mirror on the next make
local-clobber: clean
	rm -fr smlnj.fi smlnj-git *~ .rs* smlnj-conversion.tar.gz smlnj-*-git

# Force full rebuild from the remote repo on the next make.
remote-clobber: local-clobber
	rm -fr smlnj.svn smlnj-mirror smlnj-checkout smlnj-*-checkout

# Get the (empty) state of the author mapping from the first-stage stream
stubmap: smlnj.svn
	$(REPOSURGEON) "read $(READ_OPTIONS) <smlnj.svn" 'authors write >smlnj.map'

# Compare the histories of the unconverted and converted repositories at head
# and all tags.
EXCLUDE = -x CVS -x .svn -x .git
EXCLUDE += -x .svnignore -x .gitignore
headcompare: smlnj-mirror smlnj-git
	repotool compare $(EXCLUDE) smlnj-mirror smlnj-git
tagscompare: smlnj-mirror smlnj-git
	repotool compare-tags $(EXCLUDE) smlnj-mirror smlnj-git
branchescompare: smlnj-mirror smlnj-git
	repotool compare-branches $(EXCLUDE) smlnj-mirror smlnj-git
allcompare: smlnj-mirror smlnj-git
	repotool compare-all $(EXCLUDE) smlnj-mirror smlnj-git

# General cleanup and utility
clean:
	rm -fr *~ .rs* smlnj-conversion.tar.gz *.fi *.fo

# Bundle up the conversion metadata for shipping
SOURCES = Makefile smlnj.lift smlnj.map $(EXTRAS)
smlnj-conversion.tar.gz: $(SOURCES)
	tar --dereference --transform 's:^:smlnj-conversion/:' -czvf smlnj-conversion.tar.gz $(SOURCES)

dist: smlnj-conversion.tar.gz

#
# The following productions are git-specific
#

# Browse the generated git repository
gitk: smlnj-git
	cd smlnj-git; gitk --all

# Run a garbage-collect on the generated git repository.  Import doesn't.
# This repack call is the active part of gc --aggressive.  This call is
# tuned for very large repositories.
gc: smlnj-git
	cd smlnj-git; time git -c pack.threads=1 repack -AdF --window=1250 --depth=250

ignorerm:
	git filter-branch --force --index-filter \
	'git rm --cached --ignore-unmatch .gitignore' \
	--prune-empty --tag-name-filter cat -- --all
