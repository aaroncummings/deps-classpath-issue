Issue: clojure.tools.deps.alpha.extensions.pom does not build classpath
correctly for dependencies in local directories with pom.xml manifests.

Using clojure-tools-1.10.0.442.

I have a project (called 'depsproj') which depends on another project
(called 'leinproj') which was based on leiningen.  'leinproj' is in
a local directory (not a jar), which is referred to like this:
{:deps {leinproj {:local/root "/home/aaron/depstest/leinproj"}}}

'leinproj' for its manifest has a pom.xml, with sourceDirectory set to 'src'.

Running from 'depsproj', with this command:
clj -Srepro -Spath

My resulting classpath is (split into lines for readability):
    source
    /home/aaron/.m2/repository/org/clojure/clojure/1.10.0/clojure-1.10.0.jar
    /home/aaron/depstest/depsproj/src
    /home/aaron/depstest/leinproj/src/main/clojure
    /home/aaron/.m2/repository/org/clojure/spec.alpha/0.2.176/spec.alpha-0.2.176.jar
    /home/aaron/.m2/repository/org/clojure/core.specs.alpha/0.2.44/core.specs.alpha-0.2.44.jar

These two entries are incorrect:
    /home/aaron/depstest/depsproj/src
    /home/aaron/depstest/leinproj/src/main/clojure

I expect instead:
    /home/aaron/depstest/leinproj/src

The cause of this issue seems to be in this function in the
clojure.tools.deps.alpha.extensions.pom namespace:

(defmethod ext/coord-paths :pom
  [_lib {:keys [deps/root] :as coord} _mf config]
  (let [pom (jio/file root "pom.xml")
        model (read-model-file pom config)
        srcs [(.getCanonicalPath (jio/file (.. model getBuild getSourceDirectory)))
              (.getCanonicalPath (jio/file root "src/main/clojure"))]]
    (distinct srcs)))


So I have fired up tools.deps.alpha locally.  If I run this function like this
(from the clojure.tools.deps.alpha.extensions.pom namespace):
(ext/coord-paths 'leinproj {:deps/root "/home/aaron/depstest/leinproj" :deps/manifest :pom}
                 :pom {:mvn/repos maven/standard-repos})

I get this result:
("/home/aaron/depstest/tools.deps.alpha/src"
 "/home/aaron/depstest/leinproj/src/main/clojure")

The first of the two paths returned is relative to $PWD, where it
should be relative to the root of 'leinproj'.

The second of the two paths returns is relative to the correct root,
but hardcodes the source directory.  I see that this hardcoded default
path was added in commit 3d6cb2, but I'm not sure why it was added.
I'm not sure that it hurts anything (other than possibly obscuring the
first issue).

I also think that the resources paths from the pom project ought to be
included as well.

I have locally hacked this function to add the 'root' to the source directory:

(defmethod ext/coord-paths :pom
  [_lib {:keys [deps/root] :as coord} _mf config]
  (let [pom (jio/file root "pom.xml")
        model (read-model-file pom config)
        srcs (apply vector
                    (.getCanonicalPath (jio/file root (.. model getBuild getSourceDirectory)))
                    (.getCanonicalPath (jio/file root "src/main/clojure"))
                    (map #(.getCanonicalPath (jio/file root (.getDirectory %))) (.. model getBuild getResources)))]
    (distinct srcs)))

And now the results look correct:
("/home/aaron/depstest/leinproj/src"
 "/home/aaron/depstest/leinproj/src/main/clojure"
 "/home/aaron/depstest/leinproj/resources")
