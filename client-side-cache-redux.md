I'm Jon and I'm a software engineer at 100Devs. I built an app and I'm going to use a section of it that I built to explain how I made a client side user data cache with redux in order to reduce the amount of GET requests that my users needed to make. In my case, I'm looking at a section of the app where the user manages their contacts. A contacts organizer!

##### **This is a walkthrough of the entire feature but only related to redux state management, and assumes you have some minimum experience building a frontend feature with CRUD functionality connected to a backend database. It also assumes base level understanding of hooks and state management in React, and the same with Redux toolkit (RTK).**

#### In the interest of keeping it shorter and simpler, this isn't addressing adjacent concerns like data staleness, cache invalidation, and potential memory usage issues for large collections, all of which require careful handling if using a client side cache indiscriminately. This is written to explain one concept at a time.

# Step one. fun

Our serverless postgres database can be requested to provide the users entire collection of contacts all at once, *right* when the app is told by the user to load the contacts section. 

Once the contacts are received, all the following GET requests I make to find a contact are kind of a waste --- especially if no one else can make changes to them except for that user --- so I built a cache system to avoid all the extra calls to our backend services. Since the collection is never that large (no one likely has enough contacts to stress the system) I can save my postgres backend a serious amount of unnecessary work, even if it were weighed against the extra code and load the frontend adds performing this task in parallel. Nothing can be a solution for every problem, but I found this solution helpful for mine.

##### NOTE: Our code on github doesn't work without you engaging with 3rd party services and having an already setup postgres database -- but if you still just want to get your hands on the code, go to the  [ repository](https://github.com/jonathananddaryll/artemis-crm)  where this project lives to pull your copy and follow along, or you could also just continue reading, since I also include snippets.

So, here's a breakdown starting with what happens on the first load of the ContactsPage.jsx component:

[link to the file in the repo](https://github.com/jonathananddaryll/artemis-crm/blob/main/client/src/components/pages/Contacts/ContactsPage.jsx)


## Initial Database Call on Component Load

The ContactsPage serves as the main component for managing contacts in our application. When this component loads, it initiates a call to retrieve the user's contacts from our postgres database.

We use clerk.com's user authentication library, which allows us to authenticate requests to the backend using clerk's session api:

in `ContactsPage.jsx`

```jsx
import { useAuth } from  '@clerk/clerk-react';
import { getUserContactsTable } from  '../../../reducers/ContactReducer';
import { useSelector, useDispatch } from  'react-redux';
// ...other imports, stuff

export  default  function  ContactsPage() {
	const { userId, getToken } =  useAuth();
	const  dispatch  =  useDispatch();
	const  contactsLoaded  =  useSelector(state  =>  state.contact.contactsLoaded);
	// On first load, fetch all user's contacts at once and store a local copy (session)
	useEffect(() => {
	  if (!contactsLoaded) {
	    const grabSessionToken = async () => {
	      const token = await getToken();
	      dispatch(getUserContactsTable({ user_id: userId, token: 	token }));
	    };
	    grabSessionToken();
	  }
	}, [dispatch]);
	};
	return (
	// jsx
	)
```
*I've omitted all the other code that doesn't relate to this useEffect() call

### Understanding the Code

The `useEffect` hook handles side effects on React functional components. In this case, it ensures that the following code block runs when the `ContactsPage` component loads, and the Redux function `dispatch` is in the dependency array to reassure React everything is stable.

Inside of that block of code, the condition `if (!contactsLoaded)` checks if the user's contacts are already loaded by checking if `contactsLoaded` has been set to false. If so, it proceeds to fetch them.

For authentication, this initial call sends along with it a token that also contains the users id with it to be used in the next part of the sequence in redux: the asynchronous 'thunk'.

*`grabSessionToken` is an asynchronous function responsible for fetching the user's session token, used for authentication and authorization when interacting with the backend.

*`dispatch` is a function provided by Redux, used here to initiate the Redux action `getUserContactsTable` for fetching user contacts.

## Async thunks

Asynchronous thunks are for any asynchronous functions you need to define in your frontend logic, so while it's not *always* headed to an external api, in this case, it is! A quick dive into the next sequence of code:

[link to the file in the repo](https://github.com/jonathananddaryll/artemis-crm/blob/main/client/src/reducers/ContactReducer.jsx)

in `ContactsReducer.jsx`
```jsx
export const getUserContactsTable = createAsyncThunk('contacts/getUserContactsTable', async(idAndToken, thunkAPI) => {
	const config = {
		method:  'GET',
		url:  '/api/contacts',
		withCredentials:  false,
		params: {
			user_id:  idAndToken.user_id
		},
		headers: {
			'Content-Type':  'application/json',
			authorization:  `Bearer ${idAndToken.token}`
		}
		};
	try {
		const res = await axios.get(`/api/contacts/`, config);
		return res.data;
	} catch (err) {
		const  errors  =  err.response.data.errors;
		return  thunkAPI.rejectWithValue(errors);
	}
});
```

### Understanding the code  
  
In this situation, you may not need to pass tokens along like I did, but you can still get the idea of what has to happen to request a user’s contacts collection. When the response comes back fulfilled, we return to the reducer handlers the .data property of the response object. If the server rejects the request, we get errors, send those to the reducer handlers as well, and then share those with the user through the UI, too.

## Async thunk action handlers

### Handle the Pending, Rejected, Fulfilled action types

Next in the sequence of events is the .pending, .rejected, and .fulfilled action handlers. I've defined three thunk cases next, which show the three different values the server's response object may return throughout the process. 

While I've commented out my calls to toastify notification api, I just left it to share how one might also do notifications based on these states (they're found inside the extraReducers object in the Redux store)

[link to the file in the repo](https://github.com/jonathananddaryll/artemis-crm/blob/main/client/src/reducers/ContactReducer.jsx)

in `ContactReducer.jsx`
```jsx
extraReducers: builder => {
    builder.addCase(getUserContactsTable.fulfilled, (state, action) => {
      state.contactsCache = action.payload;
      state.searchResults = action.payload;
      state.contactsLoaded = true;
 //     toast.dismiss('getUserContactsTable');
    });
    builder.addCase(getUserContactsTable.pending, (state, action) => {
//      toast.loading('Loading Contacts...', {
//        toastId: 'getUserContactsTable'
//      });
    });
    builder.addCase(getUserContactsTable.rejected, (state, action) => {
//      toast.update('getUserContactsTable', {
//        render: 'Your contacts are temporarily unavailable, please try again',
//        type: toast.TYPE.ERROR,
//        isLoading: false,
//        autoClose: 2000
//      });
      action.payload.forEach(error => toast.error(error, { autoClose: 8000 }));
    });
}
```

### Understanding the code

It's only in the .fulfilled case that the state in the redux store is updated, and it updates both the `contactsCache` and `searchResults`  variables, since I wanted the user to see all their contacts on first load, and not just wait for them to search for something.

## How `contactsCache` keeps the Response Data safe in redux

The `contactsCache` array is accessible from the Redux store reducer, but it's actually never directly accessed in any UI component in our case. It could be in your app, but in ours, I designed React to check for changes to it's counterpart, `searchResults`, which is always updated at the same time. 

This way, the cache is never altered unless I have received an update to it from the backend. You can see how `searchResults` is instead used to display a filtered version of `contactsCache` here:

in `ContactsPage.jsx`:
```jsx
const searchResults = useSelector(state => state.contact.searchResults);
```

In the handler, 'action.payload' is the collection passed in from the responses .data property, and both `contactsCache` and `searchResults` get set to its value.

```jsx
state.contactsCache = action.payload;
state.searchResults = action.payload;
state.contactsLoaded = true;
toast.dismiss('getUserContactsTable');
```

##### NOTE: If you have data that gets changed while the user is still in a session, it's up to you how you decide to handle whether or not the entire cache gets updated, and when --- it get's updated by calling the same function, `getUserContactsTable`. You can trigger an update based on paused activity, lengthy activity, a timer, webhooks, whatever suits your situation.

# Step two. whoo

### CRUD activity in parallel --- Starting with READ

In this solution, once the full collection is received from the backend, we only make changes to the client side cache in parallel with backend updates $after$ the changes to the backend are validated with a successful response object to maintain one source of truth for the data. This goes for all the remaining CRUD functions as well, only now we just update one record at a time, and only send/receive one record at a time.

## 'Read'ing the Cached Data and Updating UI for the 'search' Function

The 'Read' functionality *after* the cache is initialized is actually just updating the user interface with search results pulled from an array already sitting on the client side. A search 'helper function' gets passed the local cache, and returns the results, which are then passed into the `searchResults` array. React is set up to watch changes to it for making updates (remember the useEffect() hook I mentioned at the beginning?)

in `ContactsPage.jsx`
```jsx
{!!searchResults && (
            <>
              {/* <div className={styles.flexContainer}> */}
              {searchResults.map(contact => (
                <ContactCard contactInfo={contact} key={contact.id} />
              ))}
              <div className={styles.flexFiller}></div>
              {/* </div> */}
            </>
          )}
```

So, when changes to searchResults happens in redux, the results update to the UI on the screen.

### The `searchArray` Helper Function

The `searchArray` helper function here just filters and sorts data in the `contactsCache` array based on user search criteria. You can use a library to abstract this search functionality on the client side, but be careful of how big it is! That's why I decided to write my own, even adding a custom algorithm, so the output wasn't affected as badly against misspellings and multiple word first or last names.

# Step three. wee.


### 'update'-ing the cache

At the core of the 'Update' function is the ContactForm component. This component handles the user interface for editing and updating contact information. It takes care of rendering the contact's details, providing an edit mode, and saving any changes made.

in `ContactForm.jsx`
```jsx
export default function ContactForm({ newContactStaging }) {
  async function submitUpdate(e) {
    e.preventDefault();
    // updatedValues is just an array that gets filled with key: 
    // value keys only if the input elements get interacted with,
    //  to eliminate unnecessary update requests.
    const updatedValues = [];
    if (updatedColumns.length > 0) {
      updatedColumns.map((element) => updatedValues.push(contactForm[element]));
      if (newContactStaging) {
      } else {
        const updateForm = {
          user_id: userId,
          updateWhat: updatedColumns,
          updateTo: updatedValues,
          token: await session.getToken(),
          id: contactForm.id,
        };
        dispatch(updateContact(updateForm));
        const newVersion = { ...contactInFocus, ...contactForm };
        dispatch(updateContactInFocus(newVersion));
      }
      setIsEditing(false);
    }
  }
  return (
    // ... (JSX)
  );
}
```
### Understanding the code

In the `submitUpdate` function, we first gather the updated values from the fields that have changed. If it's a new contact, we create it, and if it's an existing contact, we update it. The `updateContact` function dispatches an action to update the contact in the backend, using an async thunk. `updateContactInFocus` triggers a rerender of the UI with the new data the user just sent in the request.

## Async thunk action handler

On a successful return, the response object comes with a copy of the newly updated record in the table, which means now I know it's safe to update my client side copy:

in `ContactReducer.jsx`
```jsx
// 262
builder.addCase(updateContact.fulfilled, (state, action) => {
      // Find the contacts in the contactsCache and searchResults, then
      // update the values if they exist with the return value from
      // async thunk
      state.contactInFocus = action.payload;
      const contactsCacheIndex = SearchArray(
        `${action.payload.id}`,
        state.contactsCache,
        'id'
      )[0];
      const searchResultsIndex = SearchArray(
        `${action.payload.id}`,
        state.searchResults,
        'id'
      )[0];
      if (contactsCacheIndex === -1) {
        toast('contact not found on client side copy');
      } else if (searchResultsIndex === -1) {
        state.contactsCache[contactsCacheIndex] = action.payload;
      } else {
        state.contactsCache[contactsCacheIndex] = action.payload;
        state.searchResults[searchResultsIndex] = action.payload;
      }
      state.contactSelected = false;
      toast.dismiss('updateContact');
    });
```

### Understanding the code

In this case I'm reusing the `searchArray` helper function again, but passing in the contacts database ID number instead. Once I've found that number, I can update that specific `contactsCache` and `searchArray` redux store value with the value I just received from the backend.

# Step four. more?
## deleting

To delete a contact, the function `deleteContactstart` does a few checks to make sure the right action has to be taken - if the user was in the middle of making a contact and then deletes it, nothing needs to be done to the backend!

in `ContactForm.jsx`
```jsx
const deleteContactStart = async () => {
  if (!contactForm.id) {
    dispatch(updateContactInFocus({}));
    setContactForm(contactInFocus);
  } else {
    const formData = {
      id: contactForm.id,
      token: await session.getToken(),
    };
    dispatch(deleteContact(formData));
  }
}
```
### Understanding the code

It checks if there is already an ID assigned to the contact --- because that would mean it's definitely not a contact that's in the process of being created right then and there --- it wouldn't have an ID! If it has one, it calls the deleteContact() async thunk through dispatch, and passes the contact ID and users session token to authenticate privileges.

## The Async Thunk

Fairly simple, like the others:

in `ContactReducer.jsx`
```jsx
export const deleteContact = createAsyncThunk(
  'contacts/deleteContact',
  async (deleteRequest, thunkAPI) => {
    const headers = {
      'Content-Type': 'application/json',
      authorization: `Bearer ${deleteRequest.token}`
    };
    try {
      const res = await axios.delete(`/api/contacts/${deleteRequest.id}`, {
        headers
      });
      return res.data;
    } catch (err) {
      const errors = err.response.data.errors;
      return thunkAPI.rejectWithValue(errors);
    }
  }
);
```

## Async Thunk Handlers

Inside `ContactReducer.jsx`, the `deleteContact` async thunk receives the token and the databases ID for that contact. It sends a DELETE request to the backend and a valid response tells us we can delete it from `contactsCache` and `searchResults` safely. This is handled via the 3-state thunk handlers, so the .fulfilled logic goes like so:

in `ContactReducer.jsx`
```jsx
builder.addCase(deleteContact.fulfilled, (state, action) => {
      state.contactsCache = state.contactsCache.filter(
        contact => contact.id !== state.contactInFocus.id
      );
      state.searchResults = state.searchResults.filter(
        contact => contact.id !== state.contactInFocus.id
      );
      state.contactSelected = false;
    });
```

### Understanding the code

In this case it's much simpler, since it's an array, I just set the state to a filtered version of that state and thereby remove the deleted contact.

# Step five. it's alive!

### CREATE a contact

In the `ContactsPage.jsx`, there's an "Add" button that initiates the process of creating a new contact. It sets the `newContactStaging` state to true, and prepares the `contactInFocus` store object with default values. 

in `ContactsPage.jsx`
```jsx
const addContact = () => {
  setNewContactStaging(true);
  dispatch(
    updateContactInFocus({
      first_name: null,
      last_name: null,
      company: null,
      // ... other fields ...
    })
  );
  dispatch(updateContactSelected());
};
```

Once it's filled and the user submits the form, the `submitUpdate` function is called. It identifies the fields that have been changed to non-empty values, puts them in a form, and dispatches the `createContact` action along with the new contact details:

in `ContactForm.jsx`
```jsx
async function submitUpdate(e) {
  // ...
  if (newContactStaging) {
    const filledFields = { ... };
    const createForm = {
      ...filledFields,
      token: await session.getToken(),
    };
    dispatch(createContact(createForm));
  } else {
    // ... for an update request
}
```

Then, the `createContact` async thunk sends that request and the info to the server:

in `ContactReducer.jsx`
```jsx
export const createContact = createAsyncThunk(
  'contacts/createContact',
  async (createRequest, thunkAPI) => {
    const config = {
      headers: {
        'Content-Type': 'application/json',
        authorization: `Bearer ${createRequest.token}`
      }
    };
    try {
      const { token, ...body } = createRequest;
      const res = await axios.post('/api/contacts/', body, config);
      return res.data;
    } catch (err) {
      const errors = err.response.data.errors;
      return thunkAPI.rejectWithValue(errors);
    }
  }
);
```

and upon a successful response, updates the Redux store through the .fulfilled handler, here:

in `ContactReducer.jsx`
```jsx
builder.addCase(createContact.fulfilled, (state, action) => {
      state.newContactStaging = false;
      state.contactsCache.push(action.payload);
      state.searchResults.push(action.payload);
      state.contactInFocus = action.payload;
      state.contactSelected = false;
      toast.dismiss('createContact');
    });
```

### Understanding the code

Since the cache is an array, I just use .push() --- easy! I reset newContactStaging to false to prep the UI for other creations and that's it, state is now ready for all CRUD actions!

That's it! Done! If that's all you're here for, thanks for reading. 

# Step six. pick up sticks

## Edge Cases and Code Optimizations

In this section, we'll dive into the code snippets provided from `ContactForm.jsx`, `searchArray.js`, and `ContactReducer.jsx`. We will analyze potential areas for optimization and address edge cases in the context of the CRUD operations implemented using Redux Toolkit and client-side caching.

## Edge Cases

### Concurrent Editing

**Code:** `submitUpdate` function in `ContactForm.jsx`

The `submitUpdate` function allows users to update contact information. However, it does not handle scenarios where multiple users attempt to edit the same contact simultaneously. Implementing version control or real-time collaboration features can address this edge case.

### Large Datasets

**Code:** `searchArray.js`

The `searchArray` function is responsible for searching through cached contacts. In the case of a large dataset, searching can become slow. Implementing efficient search algorithms like binary search for sorted data or data pagination techniques could significantly enhance performance.

### Offline Support

**Code:** Various sections of `ContactReducer.jsx`

While the code doesn't explicitly address offline support, it's essential to consider how data should be handled when the application is offline. Queuing actions and syncing with the server upon reconnection are valuable strategies to account for offline scenarios.

## Code Optimizations
### Memoization

**Code:** `searchSubmit` function in `ContactsPage.jsx`

The `searchSubmit` function triggers a search whenever the user types in the search bar. Memoization techniques, like the `useMemo` hook, can be employed to optimize rendering performance by reducing unnecessary re-renders when the search parameters haven't changed.

## Conclusion

Optimizing CRUD operations in Redux with client-side caching is a complex task that could involve code optimizations and addressing potential edge cases. By focusing on these areas, you can create more efficient and responsive web applications while also ensuring data integrity and security.

As a diligent engineer, it's essential to continuously refine your code, seek new techniques, and stay updated with best practices. Sharing your experiences and insights with the development community contributes to the collective growth of knowledge.

If you found this article helpful, please consider sharing, commenting, and following me on [LinkedIn](https://linkedin.com/in/jonharveydev), and [GitHub](https://github.com/collectivenectar) for more technical content and updates to my projects. Thank you for reading!

