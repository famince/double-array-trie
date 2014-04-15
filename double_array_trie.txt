/************************************************************
Copyright (C), 20014-2016

FileName: double_array_trie.cpp
Description: Implementation of trie with doulbe array  
Version: 0.1
Function List: double array trie multi-pattern match algorithm

History:
alex.xiao        2014.3.20    version 0.1         implement

*************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <windows.h>


/*for understand the example in An "Efficient Implementation of Trie"*/
#define MACRO_FOR_DEBUG


#define MIN_CODE	2	/* Minimum numerical code */
#define MAX_CODE	255 /* Maximum numerical code */
#define BC_INC		10	/* Increment of the double-array */
#define TAIL_INC		10	/* Increment of TAIL */
#define KEY_INC		5	/* Increment of the double-array */
#define TEMP_INC	5	/* Increment of TAIL */


FILE *pKey_file = NULL;	/* Key dictionary file point*/
char *pKey = NULL;		/* Key pattern array reading from key file*/
char *pTail = NULL;		/* TAIL array*/
char *temp = NULL;	/* temp tail Buffer */
int *BC = NULL;		/* BASE and CHECK*/
int mode;	/* Flag indicating insertion, search,deletion, dump and end */
int bc_pos;	/* Current maximum index of the double-array */
int tail_pos;	/* Current maximum index of pTail */
int BC_MAX;	/* Maximum size of BASE and CHECK */
int TAIL_MAX;	/* Maximum size of pTail */
int KEY_MAX;		/* Maximum size of KEY */
int TEMP_MAX;	/* Maximum size of temp */

typedef enum da_option_e
{
	DA_INSERT_MODE = 1,	/* Indicate the insertion mode */
	DA_SEARCH_MODE = 2,	/* Indicate the search, or retrieval mode */
	DA_DELETE_MODE = 3,	/* Indicate the deletion mode */
	DA_DUMP_MODE = 4,		/* Indicate the dump mode */
	DA_EXIT_MODE = 5,		/* Indicate the end of program */
}DA_OPTION_T;


/******************************************************************************
 * The function of predefined
******************************************************************************/
char * MEM_STR(char *area_name, int * max, int n);
char *REALLOC_STR(char * area_name, char * area, int * max, int inc);
void realloc_bc(void);
char *set_list(int s);

void W_BASE (int n, int node);
void W_CHECK(int n, int node);
int X_CHECK(char * list);

void da_initialize(void);
bool da_engine (void);
void da_pattern_insert(int s, char * b);
void da_tail_insert (int s, char * a, char * b);
void da_register_separate_node (int s, char * b, int tail_pos);
void da_read_tail(int p);
void da_write_tail(char * temp, int p);
int da_change_bc (int current, int s, char * list, char ch);
void da_info_dump(int needles_num);



/******************************************************************************
 * index with each character
******************************************************************************/
int charToindex(char ch)
{
#ifdef MACRO_FOR_DEBUG
	return ch - 'a' + 2;
#else
	return ch;
#endif
}

/******************************************************************************
 * character with each index
******************************************************************************/
char indexTochar(int index)
{
#ifdef MACRO_FOR_DEBUG
	return index - 2 + 'a';
#else
	return index;
#endif
}

/****************************************************************************** 
get BASE[n] 
******************************************************************************/
int BASE (int n)	
{
	if (n > bc_pos)
	{
		return(0) ;
	}
	else
	{
		return (BC[2*n] ) ;
	}
}

/******************************************************************************
*get CHECK[n]
******************************************************************************/
int CHECK (int n)
{
	if (n > bc_pos)
	{
		return 0;
	}
	else
	{
		return BC[2*n+1];
	}
}

/******************************************************************************
return the minimum integer q such that q>0 and CHECK [ q+c ] = 0 for all 'c' in LIST,
q always starts with the value 1 and has unitary increments at analysis time.
******************************************************************************/
int X_CHECK(char * list)
{
	int i = 0, base_pos = 1, check_pos = 0;
	unsigned char ch;

	do
	{
		ch = list[i++];
		check_pos = base_pos + charToindex(ch);		/*(x-1)*/
		if (CHECK(check_pos) != 0)		/*(x-2)*/
		{
			base_pos++;
			i= 0;
			continue;
		}
	}while(list[i] != '\0');

	return(base_pos) ;
}

/******************************************************************************
set BASE[n] with node
******************************************************************************/
void W_BASE (int n, int node)
{
	while(n >= BC_MAX)
	{
		realloc_bc();
	}

	if(n > bc_pos)
	{
		bc_pos = n;
	}
	BC[2*n] = node;
}

/******************************************************************************
set CHECK[n] with node
******************************************************************************/
void W_CHECK(int n, int node)
{
	while(n >= BC_MAX)
	{
		realloc_bc();
	}

	if(n > bc_pos)
	{
		bc_pos = n;
	}

	BC[2*n + 1] = node;
}

/******************************************************************************
malloc n size of memory for *area_name.
******************************************************************************/
char * MEM_STR(char *area_name, int * max, int n)
{
	char *area;
	*max= n;

	if( (area = (char *)malloc (sizeof(char) * *max)) == NULL)
	{
		printf("%s malloc error! !\n", area_name) ;
	}

	memset(area, '\0',  sizeof(area));
	return(area) ;
}

/******************************************************************************
relloc (max + inc) size of memory for *area_name, and set the max size to (max + inc)
******************************************************************************/
char *REALLOC_STR(char * area_name, char * area, int * max, int inc)
{
	int i,pre_size;
	pre_size = *max;
	*max += inc;

	if((area= (char*) realloc(area, sizeof(char) * *max)) == NULL)
	{
		printf("%s reallot error! !\n",area_name);
		exit(-1) ;
	}

	for(i=pre_size; i<*max; i++)
	{
		area[i] = '\0';
	}

	//fprintf(stderr,"***%S realloc ***\n", area name) ;

	return(area) ;
}

/******************************************************************************
relloc double array BC[] to double size.
******************************************************************************/
void realloc_bc()
{
	int i, pre_bc = 0;
	pre_bc = BC_MAX;
	BC_MAX += BC_INC;

	if((BC = (int *)realloc(BC, sizeof(int) *2*BC_MAX) ) == NULL)
	{
		fprintf(stderr, "BC realloc error! !");
		printf("BC realloc error!!\n");
		exit(-1);
	}

	for(i = 2*pre_bc; i < 2*BC_MAX; i++)
	{
		BC[i] = 0;
	}
	fprintf(stderr, "*** BC realloc ***\n");
}

/******************************************************************************
find the branches of basis point s, and return the branches list;
******************************************************************************/
char *set_list(int s)
{
	char list[MAX_CODE-MIN_CODE+1] = {0};
	int i = 0, j = 0, t = 0;

	for (i = MIN_CODE; i < MAX_CODE-1; i++)
	{
		t = BASE (s) + i;
		if (CHECK(t) == s)
		{
			list [j++] = indexTochar(i);
		}
	}

	list [j] = '\0';

	return list;	//这里直接返回局部数组，后续优化
}

/*****************************************************************************
continues the do-while loop manipulating traversal on the reduced trie until BASE(t) becomes
negative at line (s-4), that is to say, t is a separate node number. Then, the remaining
input string (pKey+ iKey_pos+ 1) is compared with temp (the single-string STR[t] ) computed 
by da_read_tail() in order to determine whether pKey is registered or not. But only if *(pKey+iKey_pos) 
is equal to '#', then TRUE is immediately returned without accessing TAIL because STR[t] is 
empty, or null string. 

If pKey is not registered in the dictionary, then line (s-1) detects the mismatch of pKey 
on the double-array, and line (s-6) detects the mismatch on TAIL. Then, BC_lNSERT() and 
TAIL_lNSERT() are invoked for these detections, respectively, so each FALSE of lines (s-3, s-11) 
indicates that pKey was registered and TRUE of line (s-9) indicates that pKey has been already 
registered. Suppose that pKey is in the dictionary and that MODE is DA_DELETE_MODE. 
The deletion of pKey is performed at line (s-8) after confirming the existence of pKey at line (s-6).
*****************************************************************************/
bool da_engine(void)
{
	unsigned char ch;
	int iKey_pos = -1, s = 1, t = 0;
	strcat (pKey, "#");

	do
	{
		++iKey_pos ;
		ch = pKey[iKey_pos];
		t = BASE(s) + charToindex(ch);
		if (CHECK(t) != s)					/*(s-1) no same outgoing*/
		{
			if(mode == DA_INSERT_MODE)		/*(s-2)*/
			{
				da_pattern_insert(s, pKey+iKey_pos);
			}
			return FALSE;					/*(s-3)*/
		}

		if(BASE(t) < 0)					/*(s-4) old node is separate node, need take out from tail array*/
		{
			break;
		}
		s = t;
	}while(TRUE);

	if(*(pKey+iKey_pos) != '#')					/*(s-5) current pattern not end*/
	{
		da_read_tail ((-1) * BASE(t)) ;
	}

	if(*(pKey+iKey_pos) == '#' || !strcmp(temp, (pKey+iKey_pos+1)))	/*(s-6)*/
	{
		if(mode == DA_DELETE_MODE)			/*(s-7)*/
		{
			W_BASE(t, 0); W_CHECK(t, 0);	/*(s-8)*/
		}
		return TRUE;						/*(s-9)*/
	}
	else
	{
		if(mode == DA_INSERT_MODE && BASE(t) != 0)		/*(s-10)*/
		{
			da_tail_insert(t, temp, pKey+iKey_pos+1);
		}
		return FALSE;				/*(s=11)*/
	}
}

/******************************************************************************
*da_initialize() initializes the entries of BASE and CHECK by zero, and pTail by '\0',
*but, particularly, BASE[1] is initialized by 1. 
*Note that BASE[0], CHECK[0], CHECK[1] and pTail[0] are not used in this implementation
 ******************************************************************************/
void da_initialize(void)
{
	BC_MAX = BC_INC;
	KEY_MAX = KEY_INC;
	bc_pos = 1;
	tail_pos = 1;

	if((BC = (int *) malloc (sizeof (int) * 2 * BC_MAX)) == NULL) 
	{
		printf("BC malloc error!!\n");
		exit(-1);
	}

	memset(BC, 0, sizeof (int) * 2 * BC_MAX);

	//Init root node
	W_BASE(1, 1);
	W_CHECK(1, -1);

	bc_pos = 1;
	pTail = MEM_STR("tail", &TAIL_MAX, TAIL_INC);

	pTail[0] = '#';
	temp = MEM_STR("temp", &TEMP_MAX, TEMP_INC);
	pKey = MEM_STR("KEY", &KEY_MAX, KEY_INC) ;
}

/******************************************************************************
defines arc g(s, pKey_pat[0]) on the double-array, and insert the remaining of pKey_pat[] into 
double-array. if no collision, just add the remaining to pTail, otherwise should da_change_bc avoid collision.
******************************************************************************/
void da_pattern_insert(int s, char * pKey_pat)
{
	int t = 0;
	char list_s[MAX_CODE-MIN_CODE+1] = {0}, list_t[MAX_CODE-MIN_CODE+1] = {0};//, *set_list() ;

	t = BASE(s) + charToindex(*pKey_pat);
	if(CHECK(t) != 0) 					/*(b-1) has collision*/
	{
		strcpy(list_s, set_list(s));			/*(b-2)*/
		strcpy(list_t, set_list (CHECK(t))) ;	/*(b-3)*/

		if(strlen(list_s) +1 < strlen(list_t))		/*(b-4) case 4:step 4*/
		{
			s = da_change_bc(s, s, list_s, *pKey_pat);	/*(b-5)*/
		}
		else					/*(b-6)*/
		{
			s = da_change_bc(s, CHECK(t), list_t, '\0');
		}
	}

	da_register_separate_node(s, pKey_pat, tail_pos);	/*(b-7)*/
}

/******************************************************************************
store separate node pKey_pat[0] in double array and the remaining to TAIL[] from position tail_pos

defines arc g(s, b[0]) = t in CHECK[t] at line (e-1); stores position tail_pos of the single-string 
and minus sign indicating the separate node in BASE[t] at line (e-2); 
and stores the single-string into TAIL at line (e-3).
******************************************************************************/
void da_register_separate_node(int s, char * pKey_pat, int tail_pos)
{
	int t = 0;
	t = BASE(s) + charToindex(*pKey_pat);

	pKey_pat++;
	W_CHECK (t , s );				/*(e-1)*/
 	W_BASE (t, (-1) *tail_pos);	/*(e-2)A negative value indicates that the rest of the word is located in TAIL, starting at TAIL[-BASE [t]]*/

	da_write_tail (pKey_pat, tail_pos);	/*(e-3)*/
}

/******************************************************************************
da_change_bc(current, s, list, ch) modifies BASE[s] by calling the following function
X_CHECK(list) at line (c-3). 

X_CHECK()  determines the minimum index such thatbase_pos> 1 and 
CHECK(base_pos + 'c' ) = 0 for all entries 'c' in list.
******************************************************************************/
int da_change_bc (int current, int s, char * list, char ch)
{
	int i, k, old_node, new_node, old_base;
	char a_list[MAX_CODE - MIN_CODE] = {0};
	old_base = BASE(s);	/*(c-1)*/

	if(ch != '\0')			/*(c-2)*/
	{
		strcpy(a_list, list);
		i = strlen (a_list);
		a_list [i] = ch;
		a_list[i+1] = '\0';
	}

	W_BASE ( s, X_CHECK(list));					/* (c-3) case 4 : step 5*/
	i=0;

	do
	{
		old_node = old_base + charToindex(*list);	/*(c-4)*/
		new_node = BASE(s) + charToindex(*list);	/*(c-5)*/
		W_BASE(new_node, BASE(old_node));		/*(c-6) case 4 : step 6*/
		W_CHECK(new_node, s) ;					/*(c-7)*/

		//map all node based as old_node to new_node
		if(BASE(old_node) > 0)						/*(c-8)*/
		{
			k = BASE(old_node)+1;
			while (k-BASE(old_node) < MAX_CODE-MIN_CODE && k < bc_pos)	/*(c-9)*/
			{
				if(CHECK(k) == old_node)
					W_CHECK(k, new_node) ;			/*(c-10) case 4 :step 7*/
				++k ;
			}
		}

		if(current != s && old_node == current)		/*(c-11)*/
		{
			current = new_node;
		}
		W_BASE(old_node, 0);				/*(c-12) case 4 :step 8*/
		W_CHECK(old_node, 0);
		list++;
	}while(*list != '\0');

	return current;			/*(c-13)*/
}

/******************************************************************************
For the current node number s and for the remaining input string pKey,
the function da_tail_insert(s, pTail, pKey) defines arc g(s, pKey[0]) = t  on the double-array 
and stores the single-string STR[t]=b[1]b[2]... in TAIL.

The while-loop at lines from (t-3) to (t-6) appends pTail sequence of arcs 
for the longest prefix pTail[0]pTail[1]...pTail[length-1] of strings pTail and pKey. 

Lines (t-8, t-9) defines arcs labelled pTail[length] and pKey[length] are stored in the double-array.

The function separate() invoked at line (t-8, t-9) defines arcs such that g(s,a[length]) = t
and g(s,b[length]) = t' on the double-array, and stores the both remaining strings
a+length+ 1 and b+length+ 1 for separate nodes t and t' into TAIL.
******************************************************************************/
void da_tail_insert (int s, char * pTail, char * pKey)
{
	char list[40] = {0};
	unsigned char ch;
	int i = 0, length = 0, t = 0, old_tail_pos = 0;

	old_tail_pos = (-1) * BASE(s);			/*(t-1)*/
	while (pTail[length] == pKey[length] )	/*(t-2)*/
	{
		length++;		//calculate the longest same prefix length
	}

	//first init the same prefix node
	while (i < length)						/*(t -3)*/
	{
		ch = pTail[i++];
		list [0] = ch;
		list [i] = '\0';
		W_BASE (s, X_CHECK (list));		/*(t -4)*/
		t = BASE(s) + charToindex(ch);		/*(t-5)*/

		W_CHECK(t, s) ;					/*(t-6)*/
		s = t;
	}

	//second init respective separate node, and save differrent suffix to TAIL
	list[0] = pTail[length];					/*(t-7)*/
	list[1] = pKey[length];
	list[2] = '\0';
	W_BASE(s, X_CHECK(list));				/*(t-8)*/

	da_register_separate_node(s, pTail+length, old_tail_pos) ;	/*(t-9)*/
	da_register_separate_node(s, pKey+length, tail_pos);		/*(t-10*/
}

/******************************************************************************
copies the requested single string to temp[] start from p.
******************************************************************************/
void da_read_tail(int p)
{
	int i = 0;

	while(pTail[p] != '#')
	{
		temp[i++] = pTail[p++];
	}
	temp[i++] = '#';
	temp[i] = '\0';
}

/******************************************************************************
stores the single-string into the requested address p of pTail
******************************************************************************/
void da_write_tail(char * temp, int iTail_old_pos)
{
	int i = 0, tail_index = 0;
	tail_index = iTail_old_pos;

	while((iTail_old_pos + strlen(temp)) >= TAIL_MAX-1)
	{
		pTail = REALLOC_STR("tail", pTail, &TAIL_MAX, TAIL_INC) ;
	}

	while(*(temp+i) != '\0')
	{
		pTail[tail_index++] = *(temp + i++);
	}

	if( (iTail_old_pos + i ) > tail_pos)
	{
		tail_pos = iTail_old_pos + i;
	}
}

/******************************************************************************
displays the sizes of the doubIe-array and tail, and all values of the double array 
and pTail are included if DA_DUMP_MODE is selected.
******************************************************************************/
void da_info_dump(int needles_num)
{
	int i = 0, bc_empty = 0;

	for(i=0; i <= bc_pos; ++i)
	{
		if(BASE(i) == 0 && CHECK(i) ==0)
		{
			bc_empty++;
		}
	}

	if(mode == DA_DUMP_MODE) 
	{
		printf("\n");
		printf("  Index |  BASE  |  CHECK |\n");
		for(i=0; i <= bc_pos; ++i)
		{
			printf("%7d |%7d | %7d|\n", i, BASE(i), CHECK(i));
		}

		for(i=0; i < tail_pos; ++i)
		{
			printf("%d%c|", i, pTail[i]);
		}
		printf("\n");
	}
	printf("Total number of keys = %d\n", needles_num-1);
	printf("bc_pos = %d\n", bc_pos);
	printf("bc_empty = %d\n", bc_empty);
	printf("tail_pos = %d\n", tail_pos);
}

/******************************************************************************
show the options which can be selected
******************************************************************************/
void da_function_option(void)
{
	char key_name[30] = {0};
	printf("**************************************************\n");
	printf("************ double array trie ******************* \n");
	printf(" 1. Insert  2. Search  3. Delete  4. Dump  5. End \n");
	printf("**************************************************\n");
	scanf("%d%*c", &mode) ;

	if(mode == DA_EXIT_MODE)
	{
		exit(0) ;
	}

	if(mode != DA_DUMP_MODE)
	{
		printf("key_file = ");
		scanf("%s%*c", key_name) ;
		pKey_file = fopen(key_name, "r") ;
		if(pKey_file == NULL)
		{
			printf("patterns file not exist or open failed !\n");
			exit(0);
		}
	}
}


void main(void)
{
	int c = 0, i = 0, needles_num = 0;

	da_initialize();

	for(;;)
	{
		da_function_option();
		needles_num = 1;
		while( (c = getc(pKey_file) ) != EOF)
		{
			if(c != '\n')
			{
				while(i >= KEY_MAX-2)
				{
					pKey = REALLOC_STR("KEY", pKey, &KEY_MAX, KEY_INC);
					temp = REALLOC_STR("temp", temp, &TEMP_MAX, TEMP_INC);
				}
				pKey[i++] = c;
				continue;
			}
			pKey[i] = '\0';
			i = 0;

			switch(mode)
			{
				case DA_INSERT_MODE:
					if(da_engine() == FALSE)
					{
						printf("%d : %s is inserted successed !\n", needles_num++, pKey);
					}
					else
					{
						printf("%d:%s is already in your dictionary\n", needles_num++, pKey);
					}
					break;

				case DA_SEARCH_MODE:
					if(da_engine() == TRUE)
					{
						printf("%d:%s is searched\n", needles_num++, pKey);
					}
					else
					{
						printf("%d:%s is mismatch\n", needles_num++, pKey);
					}
					break;

				case DA_DELETE_MODE:
					if(da_engine() == TRUE)
					{
						printf("%d:%s is deleted\n", needles_num++, pKey);
					}
					else
					{
						printf("%d:%s is not in your dictionary\n", needles_num++, pKey) ;
					}
					break;

				case DA_DUMP_MODE: 
					break;
				default: 
					exit(0);
			}
		}

		da_info_dump(needles_num) ;
		fclose(pKey_file) ;
	}
}


