# Test id

`testID` and `accessibilityLabel` properties serve as locator for e2e tests.
They should be defined for all actionable components (`<Button>`, `<Input>` etc.).

`getTestIdProps({ parentId, baseId, index, id }: TestIdValues): TestIdProps` function from `utils/index.ts` helps to define these props for components.

Where `TestIdValues` and `TestIdProps` stands for:

```typescript
type TestIdValues = {
  parentId?: string;
  baseId: string;
  index?: number;
  id?: string | number;
};

type TestIdProps = {
  testID: string;
  accessibilityLabel: string;
};
```

- `parentId` - test id passed from outer component
- `baseId` - test id specific for the current component
- `index` - used to distinguish components of the same type in an array (should NOT be used inside basic components)
- `id` - used to specify what exactly component is responsible for (should NOT be used inside basic components)

Basically, `getTestId()` concatenate passed values and converts into kebab-case. 
On the other hand, `getTestIdProps()` returns an object with props we need for locators.

```js
const parentId = 'parent';
const baseId = 'base';
const index = 0;
const id = 'id';

getTestId({ parentId, baseId, index, id }) === 'parent-base-0-id';
getTestIdProps({ parentId, baseId, index, id }) === { testID: 'parent-base-0-id', accessibilityLabel: 'parent-base-0-id' };
```

More test cases are described in related [tests](../src/utils/__tests__/index.spec.tsx).

## Rules of thumb

`baseId` - value in kebab-style, usually name of file. This constant should be defined for all components, regardless is it a container or basic component.
It is required to make all components' test ids unique among the app.

- `/src/screens/HomeScreen.tsx`

  ```js
  const baseId = 'home-screen';

  const HomeScreen = () => {
    return (
      ...
  ```

- `/src/components/Button/index.tsx`

  ```js
  const baseId = 'button';

  const Button = () => {
    return (
      ...
  ```

`parentId` - is a property passed to child component which is combination of `parentId` (if exist),
current `baseId`, `index` (if map) and `id` (if more distinguish is required) using `getTestId`.

- `/src/components/SubscribeForm/index.tsx`

  ```js
  const baseId = 'subscribe-form';

  const SubscribeForm = (props) => {
    const { onSubmit, parentId } = props;
    return (
      <>
        <Button
          title="submit"
          onPress={onSubmit}
          parentId={getTestId({ parentId, baseId, id: 'submit' })}
        />
        <TouchableOpacity
          onPress={onSubmit}
          {...getTestIdProps({ parentId, baseId, id: 'cancel' })}
        >
          <Text>Cancel</Text>
        </TouchableOpacity>
      </>
    );
  };
  ```

**NOTE:** we set `testID` and `accessibilityLabel` only for _atom_ components (`<TouchableOpacity/>`, `<Input/>`, `<Button/>`, etc. that
is imported from `react-native` or any other library that we don't have control of). Otherwise, use `parentId`
to pass ids down to atom component.

## Example

Consider we have to cover `SignIn` screen with testIDs, where `<Input/>` and `<Button/>` our custom components.

```js
const SignInScreen = () => {
  return (
    <>
      <Input placeholder="Email" onChangeText={setEmail} />
      <Input placeholder="Password" onChangeText={setPassword} />
      <Button title="Sign in" onPress={onSignInClick} />
    </>
  );
};
```

1. Add `baseId` for the screen and pass it to `<Input/>` and `<Button/>` through `parentId`:

   `/src/screens/SignInScreen/index.tsx`

   ```js
   const baseId = 'sign-in-screen'; // Added

   const SignInScreen = () => {
     return (
       <>
         <Input
           placeholder="Email"
           onChangeText={setEmail}
           parentId={getTestId({ baseId, id: 'email' })} // Added
         />
         <Input
           placeholder="Password"
           onChangeText={setPassword}
           parentId={getTestId({ baseId, id: 'password' })} // Added
         />
         <Button
           title="Sign in"
           onPress={onSignInClick}
           parentId={getTestId({ baseId, id: 'submit' })} // Added
         />
       </>
     );
   };
   ```

1. Add `baseId` to `<Input/>` and `<Button/>` components and set `testID` for their atom components:

   `/src/components/Input/index.tsx`

   ```js
   const baseId = 'input'; // Added

   const Input = (props) => {
     const { parentId } = props; // Added
     return (
       <InputContainer {...props} isFocused={isFocused}>
         <StyledInput
           {...props}
           onBlur={() => setFocused(false)}
           onFocus={() => setFocused(true)}
           {...getTestIdProps({ parentId, baseId })} // Added
         />
       </InputContainer>
     );
   };
   ```

   `/src/components/Button/index.tsx`

   ```js
   const baseId = 'button'; // Added

   const Button = (props) => {
     const {
       title,
       disabled,
       onPress,
       outline,
       danger,
       parentId, // Added
       ...rest
     } = props;
     const activeOpacity = disabled ? 1 : 0.5;

     return (
       <StyledTouchableOpacity
         {...rest}
         outline={outline}
         danger={danger}
         activeOpacity={activeOpacity}
         disabled={disabled}
         onPress={onPress}
         {...getTestIdProps({ parentId, baseId })} // Added
       >
         <StyledText
           disabled={disabled}
           outline={outline}
           danger={danger}
           {...getTestIdProps({ parentId, baseId, id: 'title' })} // Added
         >
           {title}
         </StyledText>
       </StyledTouchableOpacity>
     );
   };
   ```

1. As a result we have next testIDs and accessibilityLabels:
   - `sign-in-screen-input-email`
   - `sign-in-screen-input-password`
   - `sign-in-screen-submit-button`
   - `sign-in-screen-submit-button-title`

## TestID in array

Consider we have stocks list screen.

1. Define `<StockItem/>` component

   ```js
   const baseId = 'stock-item';

   const StockItem = (props) => {
     const { title, price, onPress, parentId } = props;
     return (
       <TouchableOpacity
         onPress={onPress}
         {...getTestIdProps({ parentId, baseId })}
       >
         <Text {...getTestIdProps({ parentId, baseId, id: 'title' })}>
           {title}
         </Text>
         <Text {...getTestIdProps({ parentId, baseId, id: 'price' })}>
           {price}
         </Text>
       </TouchableOpacity>
     );
   };
   ```

1. Use this component in the `<MarketScreen/>`

   ```js
   const baseId = 'market-screen';

   const MarketScreen = (props) => {
     const { title, price, onPress, parentId } = props;
     const stocks = ['AAPL', 'TSLA', 'GOOGL'];
     return (
       <>
         <Text>Current stocks:</Text>

         {stocks.map((stock, index) => (
           <StockItem
             title={stock}
             onPress={openDetails(stock)}
             parentId={getTestId({ baseId, index })}
           />
         ))}

         <Button
           title="Refresh"
           parentId={getTestId({ baseId, id: 'refresh' })}
         />
       </>
     );
   };
   ```

1. As a result we have next testIDs and accessibilityLabels:
   - `market-screen-0-stock-item`
   - `market-screen-0-stock-item-title`
   - `market-screen-0-stock-item-price`
   - `market-screen-1-stock-item`
   - `market-screen-1-stock-item-title`
   - `market-screen-1-stock-item-price`
   - `market-screen-2-stock-item`
   - `market-screen-2-stock-item-title`
   - `market-screen-2-stock-item-price`
   - `market-screen-refresh-button`
