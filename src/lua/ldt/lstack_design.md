-- Large Stack Object (LSO or LSTACK) Operations
-- Track the data and iteration of the last update.
local MOD="lstack_2013_10_07.a";

-- This variable holds the version of the code (Major.Minor).
-- We'll check this for Major design changes -- and try to maintain some
-- amount of inter-version compatibility.
local G_LDT_VERSION = 1.1;

-- ======================================================================
-- || GLOBAL PRINT ||
-- ======================================================================
-- Use this flag to enable/disable global printing (the "detail" level
-- in the server).
-- ======================================================================
local GP=true; -- Leave this ALWAYS true (but value seems not to matter)
local F=true; -- Set F (flag) to true to turn ON global print
local E=true; -- Set E (ENTER/EXIT) to true to turn ON Enter/Exit print
local B=true; -- Set B (Banners) to true to turn ON Banner Print

-- ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- <<  LSTACK Main Functions >>
-- ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- The following external functions are defined in the LSTACK module:
--
-- (*) Status = push( topRec, ldtBinName, newValue, userModule )
-- (*) Status = push_all( topRec, ldtBinName, valueList, userModule )
-- (*) List   = peek( topRec, ldtBinName, peekCount ) 
-- (*) List   = pop( topRec, ldtBinName, popCount ) 
-- (*) List   = scan( topRec, ldtBinName )
-- (*) List   = filter( topRec, ldtBinName, peekCount,userModule,filter,fargs)
-- (*) Status = destroy( topRec, ldtBinName )
-- (*) Number = size( topRec, ldtBinName )
-- (*) Map    = get_config( topRec, ldtBinName )
-- (*) Status = set_capacity( topRec, ldtBinName, new_capacity)
-- (*) Status = get_capacity( topRec, ldtBinName )

-- ======================================================================
-- LSTACK TODO LIST:
-- Future: Short, Medium and Long Term
-- Priority: High, Medium and Low
-- Difficulty: High, Medium and Low
-- ======================================================================
-- ACTIVITIES LIST:
-- (*) DONE: (Started: July 23, 2012)
--   + Release Cold Storage:  Release an entire Cold Dir Page (with the list
--     of subrec digests) on addition of a new Cold-Head, when the Cold Dir
--     Count > Max.
--   + Also, change the Cold Dir Pages to be doubly linked rather than singly
--     linked.
--   + Add a Cold Dir TAIL pointer in addition to the HEAD pointer -- so that
--     releasing the oldest Dir Rec Page (and its children) is quicker/easier.
--
-- (*) DONE: (Started: July 23, 2012)
--   + Implement the subrecContext (subrec management) here in LSTACK that
--     we have in LLIST.  Name them all ldt_subrecXXXX() because they
--     will become ldt common methods.
--
-- (*) HOLD: (Started: July 23, 2012)
--   + Switch to ldt_common.lua for the newly common functions.
--   + Put on hold until next release work.
--
-- (*) IN-DONE: (Started: July 23, 2012)
--   + Added new method: lstack_set_storage_limit(), which limits peek sizes
--     and also sets/resets the storage parameter values so that item counts
--     over the size limits (Hot, Warm or Cold list) will discard old data.
--
-- (*) IN-PROGRESS: (Started: July 25, 2012)
--   + Add new "crec_release" method that takes a LIST of digests and
--     releases the storage.  Raj will fill in the C code on the server
--     side does the delete.
--
-- TODO:
-- (+) HOLD:  Trim is less important than "set_storage_limit()", and is
--     more expensive.
--     Implement Trim (release LDR pages from Warm/Cold List)
--     stack_trim(): Must release storage before record delete.
--     Notice that this is lower priority now that we're going to look to
--     the Cold List Storage Release to deal with storage reclaimation
--     from lstack eviction.
--     F:S, P:M, D:M
-- (+) Add a LIMIT value to the control map -- and when we are a page over
--     then release the page (ideally -- do not slow down transaction to
--     do this.  Can we add a task to a cleanup thread for this?)
--     Answers:
--     (1) We set a new MAP value (M_StoreLimit) so that we never read
--     past the end.
--     (2) We check the storage state on ColdList Insert to see if we're
--     past the end -- and if so -- we release any LDRs (and thus Cold Dirs)
--     that are holding "Frozen data" that is beyond the end. Note that
--     we could eventually save "Frozen Data" in a file for permanent
--     archive, if we so desired.  That could also be done for a background
--     task.
--     (3) Define the SearchPath object for LSTACK to find a position,
--     then let other functions do things relative to that position, such
--     as release storage.
-- (+) NOTICE: Position Calc is LIST MODE ONLY::
--     : Must be extended for BINARY MODE.
--   These must be kept current:  M_WarmTopEntryCount, M_WarmTopByteCount      
--   They will replace WarmTopFul with an actual count.
--
-- ======================================================================
-- DONE:
-- (*) Implement LDT Remove -- remove the ESR and the Bin Contents, and
--     then let the NSUP/Defrag mechanism clean up the subrecs.
-- (*) ldtInitPropMap() method
-- (*) Init the record Prop Bin on first LDT Create
-- (*) Add Bin Flags, and Record Types (record.set_flags(), set_type())
-- (*) Add ESR:  Create the ESR on first SubRec Create
-- (*) Add a Record-level Hidden Bin (holds Record Version Info, LDT Count)
-- (*) Switch to ldtCtrl (PropMap, ldtMap):: Common Property Map
-- (*) LStack Peek with filters.
-- (*) LStack Peek with Transform/Untransform
-- (*) LStack Packages to define sets of parameters
-- (*) Main LStack Functions (push, peek:: Simple, Complex)
--
-- ======================================================================
-- Additional lstack documentation may be found in: lstack_design.lua.
-- ======================================================================
-- LSTACK Design and Type Comments (aka Large Stack Object, or LSO).
-- The lstack type is a member of the new Aerospike Large Type family,
-- Large Data Types (LDTs).  LDTs exist only on the server, and thus must
-- undergo some form of translation when passing between client and server.
--
-- LSTACK is a server side type that can be manipulated ONLY by this file,
-- lstack.lua.  We prevent any other direct manipulation -- any other program
-- or process must use the lstack api provided by this program in order to
--
-- An LSTACK value -- stored in a record bin -- is represented by a Lua MAP
-- object that comprises control information, a directory of records
-- (for "warm data") and a "Cold List Head" ptr to a linked list of directory
-- structures that each point to the records that hold the actual data values.
--
-- LSTACK Functions Supported (Note switch to lower case)
-- (*) create: Create the LSTACK structure in the chosen topRec bin
-- (*) push: Push a user value (AS_VAL) onto the stack
-- (*) create_and_push: Push a user value (AS_VAL) onto the stack
-- (*) peek: Read N values from the stack, in LIFO order
-- (*) peek_then_filter: Read N values from the stack, in LIFO order
-- (*) trim: Release all but the top N values.
-- (*) remove: Release all storage related to this lstack object
-- (*) config: retrieve all current config settings in map format
-- (*) size: Report the NUMBER OF ITEMS in the stack.
-- (*) subrec_list: Return the list of subrec digests
--
-- |||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- Visual Depiction
-- |||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- In a user record, the bin holding the Large Stack Object (LSTACK) is
-- referred to as an "LSTACK" bin. The overhead of the LSTACK value is 
-- (*) LSTACK Control Info (~70 bytes)
-- (*) LSTACK Hot Cache: List of data entries (on the order of 100)
-- (*) LSTACK Warm Directory: List of Aerospike Record digests:
--     100 digests(250 bytes)
-- (*) LSTACK Cold Directory Head (digest of Head plus count) (30 bytes)
-- (*) Total LSTACK Record overhead is on the order of 350 bytes
-- NOTES:
-- (*) In the Hot Cache, the data items are stored directly in the
--     cache list (regardless of whether they are bytes or other as_val types)
-- (*) In the Warm Dir List, the list contains aerospike digests of the
--     LSTACK Data Records (LDRs) that hold the Warm Data.  The LDRs are
--     opened (using the digest), then read/written, then closed/updated.
-- (*) The Cold Dir Head holds the Aerospike Record digest of a record that
--     holds a linked list of cold directories.  Each cold directory holds
--     a list of digests that are the cold LSTACK Data Records.
-- (*) The Warm and Cold LSTACK Data Records use the same format -- so they
--     simply transfer from the warm list to the cold list by moving the
--     corresponding digest from the warm list to the cold list.
-- (*) Record types used in this design:
-- (1) There is the main record that contains the LSTACK bin (LSTACK Head)
-- (2) There are LSTACK Data "Chunk" Records (both Warm and Cold)
--     ==> Warm and Cold LSTACK Data Records have the same format:
--         They both hold User Stack Data.
-- (3) There are Chunk Directory Records (used in the cold list)
--
-- (*) How it all connects together....
-- (+) The main record points to:
--     - Warm Data Chunk Records (these records hold stack data)
--     - Cold Data Directory Records (these records hold ptrs to Cold Chunks)
--
-- (*) We may have to add some auxilliary information that will help
--     pick up the pieces in the event of a network/replica problem, where
--     some things have fallen on the floor.  There might be some "shadow
--     values" in there that show old/new values -- like when we install
--     a new cold dir head, and other things.  TBD
--
-- +-----+-----+-----+-----+----------------------------------------+
-- |User |User |o o o|LSO  |                                        |
-- |Bin 1|Bin 2|o o o|Bin 1|                                        |
-- +-----+-----+-----+-----+----------------------------------------+
--                  /       \                                       
--   ================================================================
--     LSTACK Map                                              
--     +-------------------+                                 
--     | LSO Control Info  | About 20 different values kept in Ctrl Info
--     |...................|
--     |...................|< Oldest ................... Newest>            
--     +-------------------+========+========+=======+=========+
--     |<Hot Entry Cache>  | Entry 1| Entry 2| o o o | Entry n |
--     +-------------------+========+========+=======+=========+
--     |...................|HotCache entries are stored directly in the record
--     |...................| 
--     |...................|WarmCache Digests are stored directly in the record
--     |...................|< Oldest ................... Newest>            
--     +-------------------+========+========+=======+=========+
--     |<Warm Digest List> |Digest 1|Digest 2| o o o | Digest n|
--     +-------------------+===v====+===v====+=======+====v====+
--  +-<@>Cold Dir List Head|   |        |                 |    
--  |  +-------------------+   |        |                 |    
--  |                    +-----+    +---+      +----------+   
--  |                    |          |          |     Warm Data(WD)
--  |                    |          |          |      WD Rec N
--  |                    |          |          +---=>+--------+
--  |                    |          |     WD Rec 2   |Entry 1 |
--  |                    |          +---=>+--------+ |Entry 2 |
--  |                    |      WD Rec 1  |Entry 1 | |   o    |
--  |                    +---=>+--------+ |Entry 2 | |   o    |
--  |                          |Entry 1 | |   o    | |   o    |
--  |                          |Entry 2 | |   o    | |Entry n |
--  |                          |   o    | |   o    | +--------+
--  |                          |   o    | |Entry n |
--  |                          |   o    | +--------+
--  |                          |Entry n | "LDR" (LSTACK Data Record) Pages
--  |                          +--------+ [Warm Data (LDR) Chunks]
--  |                                            
--  |                                            
--  |                           <Newest Dir............Oldest Dir>
--  +-------------------------->+-----+->+-----+->+-----+-->+-----+-+
--   (DirRec Pages DoubleLink)<-+Rec  |<-+Rec  |<-+Rec  | <-+Rec  | V
--    The cold dir is a linked  |Chunk|  |Chunk|  |Chunk| o |Chunk|
--    list of dir pages that    |Dir  |  |Dir  |  |Rec  | o |Dir  |
--    point to LSO Data Records +-----+  +-----+  +-----+   +-----+
--    that hold the actual cold [][]:[]  [][]:[]  [][]:[]   [][]:[]
--    data (cold chunks).       +-----+  +-----+  +-----+   +-----+
--                               | |  |   | |  |   | |  |    | |  |
--    LDRS (per dir) have age:   | |  V   | |  V   | |  V    | |  V
--    <Oldest LDR .. Newest LDR> | |::+--+| |::+--+| |::+--+ | |::+--+
--    As "Warm Data" ages out    | |::|Cn|| |::|Cn|| |::|Cn| | |::|Cn|
--    of the Warm Dir List, the  | V::+--+| V::+--+| V::+--+ | V::+--+
--    LDRs transfer out of the   | +--+   | +--+   | +--+    | +--+
--    Warm Directory and into    | |C2|   | |C2|   | |C2|    | |C2|
--    the cold directory.        V +--+   V +--+   V +--+    V +--+
--                               +--+     +--+     +--+      +--+
--    The Warm and Cold LDRs     |C1|     |C1|     |C1|      |C1|
--    have identical structure.  +--+     +--+     +--+      +--+
--                                A        A        A         A    
--                                |        |        |         |
--     [Cold Data (LDR) Chunks]---+--------+--------+---------+
--
--
-- The "Hot Entry Cache" is the true "Top of Stack", holding roughly the
-- top 50 to 100 values.  The next level of storage is found in the first
-- Warm dir list (the last Chunk in the list).  Since we process stack
-- operations in LIFO order, but manage them physically as a list
-- (append to the end), we basically read the pieces in top down order,
-- but we read the CONTENTS of those pieces backwards.  It is too expensive
-- to "prepend" to a list -- and we are smart enough to figure out how to
-- read an individual page list bottom up (in reverse append order).
--
-- We don't "age" the individual entries out one at a time as the Hot Cache
-- overflows -- we instead take a group at a time (specified by the
-- HotCacheTransferAmount), which opens up a block of empty spots. Notice that
-- the transfer amount is a tuneable parameter -- for heavy reads, we would
-- want MORE data in the cache, and for heavy writes we would want less.
--
-- If we generally pick half (e.g. 100 entries total, and then transfer 50 at
-- a time when the cache fills up), then half the time the inserts will affect
-- ONLY the Top (LSTACK) record -- so we'll have only one Read, One Write 
-- operation for a stack push.  1 out of 50 will have the double read,
-- double write, and 1 out of 10,000 (or so) will have additional
-- IO's depending on the state of the Warm/Cold lists.
-- Notice ALSO that when we use a coupled Namespace for LDTs (main memory
-- for the top records and SSD for the subrecords), then 49 out of 50
-- writes and small reads will have ZERO I/O cost -- since it will be
-- contained in the main memory record.
--
-- NOTES:
-- Design, V3.x.  For really cold data -- things out beyond 50,000
-- elements, it might make sense to just push those out to a real disk
-- based file (to which we could just append -- and read in reverse order).
-- If we ever need to read the whole stack, we can afford
-- the time and effort to read the file (it is an unlikely event).  The
-- issue here is that we probably have to teach Aerospike how to transfer
-- (and replicate) files as well as records.
--
-- Design, V3.x. We will need to limit the amount of data that is held
-- in a stack. We've added "StoreLimit" to the ldtMap, as a way to limit
-- the number of items.  Note that this can be used to limit both the
-- storage and the read amounts.
-- One way this could be used is to REUSE a cold LDR page when an LDR
-- page is about to fall off the end of the cold list.  However, that
-- must be considered carefully -- as the time and I/O spent messing
-- with the cold directory and the cold LDR could be a performance hit.
-- We'll have to consider how we might age these pages out gracefully
-- if we can't cleverly reuse them (patent opportunity here).
-- ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
-- REMEMBER THAT ALL INSERTS ARE INTO HOT LIST -- and transforms are done
-- there.  All UNTRANSFORMS are done reading from the List (Hot List or
-- warm/cold Data Page List).  Notice that even though the values may be
-- transformed (compacted into) bytes, they are still just inserted into
-- the hot list, we don't try to pack them into an array;
-- that is done only in the warm/cold pages (where the benefit is greater).
-- 
-- Read Filters are applied AFTER the UnTransform (bytes and list).
--
-- NOTE: New changes with V4.3 to Push and Peek.
-- (*) Stack Push has an IMPLICIT transform function -- which is defined
--     in the create spec.  So, the two flavors of Stack Push are now
--     + lstack_push(): with implicit transform when defined
--     + lstack_create_and_push(): with the ability to create as
--       needed -- and with the supplied create_spec parameter.
-- (*) Stack Peek has an IMPLICIT UnTransform function -- which is defined
--     in the create spec.  So, the two flavors of Stack Peek are now
--     + lstack_peek(): with implicit untransform, when defined in create.
--     + lstack_peek_then_filter(): with implicit untransform and a filter
--       to act as an additional query mechanism.
--
-- On Create, a Large Stack Object can be configured with a Transform function,
-- to be used on storage (push) and an UnTransform function, to be used on
-- retrieval (peek).
-- (*) stack_push(): Push a user value (AS_VAL) onto the stack, 
--     calling the Transform on the value FIRST to transform it before
--     storing it on the stack.
-- (*) stack_peek_then_filter: Retrieve N values from the stack, and for each
--     value, apply the transformation/filter UDF to the value before
--     adding it to the result list.  If the value doesn't pass the
--     filter, the filter returns nil, and thus it would not be added
--     to the result list.
-- ======================================================================
-- Aerospike Server Functions:
-- ======================================================================
-- Aerospike Record Functions:
-- status = aerospike:create( topRec )
-- status = aerospike:update( topRec )
-- status = aerospike:remove( rec ) (not currently used)
--
--
-- Aerospike SubRecord Functions:
-- newRec = aerospike:create_subrec( topRec )
-- rec    = aerospike:open_subrec( topRec, childRecDigest)
-- status = aerospike:update_subrec( childRec )
-- status = aerospike:close_subrec( childRec )
-- status = aerospike:remove_subrec( subRec )  
--
-- Record Functions:
-- digest = record.digest( childRec )
-- status = record.set_type( topRec, recType )
-- status = record.set_flags( topRec, binName, binFlags )
-- ======================================================================
-- ======================================================================
-- ======================================================================